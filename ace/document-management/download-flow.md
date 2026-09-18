# 文档下载与预览流程

## 概述

RAGFlow 提供多种文档资源获取端点，支持原始文档下载、浏览器内预览、缩略图批量获取、图片服务以及沙盒工件访问。所有端点都通过 MinIO/S3 存储后端统一读取文件，但在响应头处理、权限校验和 MIME 类型判断上有不同策略。

**核心端点**:
- **下载**: `GET /datasets/{id}/documents/{doc_id}` 与 `GET /documents/{doc_id}` — 强制下载为附件
- **预览**: `GET /documents/{doc_id}/preview` — 浏览器内渲染（HTML/SVG 强制下载避免 XSS）
- **缩略图批量**: `GET /thumbnails?doc_ids=...` — 返回 ID→URL 映射
- **图片服务**: `GET /documents/images/{image_id}` — 直接返回图片字节流
- **沙盒工件**: `GET /documents/artifact/{filename}` — 代码执行沙盒生成的文件

---

## 流程 1: 文档下载（强制附件）

**端点**: 
- `GET /datasets/{dataset_id}/documents/{document_id}` — 带知识库 ID 路径
- `GET /documents/{document_id}` — 仅文档 ID（内部查询知识库）

两个端点实现几乎相同，区别仅在于第一个需校验知识库所有权。

### 执行步骤

**步骤 1: 多层权限校验**

```python
# 端点 1: 校验知识库所有权 + 文档可访问性
if not KnowledgebaseService.accessible(kb_id=dataset_id, user_id=current_user.id):
    return get_data_error_result(message="document not found")
if not DocumentService.accessible(document_id, current_user.id):
    return get_data_error_result(message="document not found")
doc = DocumentService.query(kb_id=dataset_id, id=document_id)

# 端点 2: 仅校验文档可访问性
if not DocumentService.accessible(document_id, current_user.id):
    return get_data_error_result(message="document not found")
doc = DocumentService.query(id=document_id)
```

**防御设计**: 错误消息统一为 "document not found"（权限不足与文档不存在不可区分），避免跨租户 ID 枚举攻击。

**步骤 2: 解析存储地址**

`File2DocumentService.get_storage_address` 返回 `(bucket, object_key)` 元组：

```python
doc_id, doc_location = File2DocumentService.get_storage_address(doc_id=document_id)
# doc_id = kb_id (作为 bucket)
# doc_location = 文件在 MinIO 中的对象路径
```

**地址来源逻辑**（在 `file2document_service.py` 中）:
1. 查询 `File2Document` 关联表获取 `file_id`
2. 从 `File` 表获取 `parent_id`（bucket）与 `location`（object key）
3. 如果 `source_type` 为 `LOCAL` 或空，返回 `File` 的存储地址
4. 否则回退到 `Document.kb_id` 与 `Document.location`（用于非本地上传的文档，如连接器）

**步骤 3: 存储后端读取**

```python
file_stream = settings.STORAGE_IMPL.get(doc_id, doc_location)
if not file_stream:
    return construct_json_result(message="This file is empty.", code=RetCode.DATA_ERROR)
file = BytesIO(file_stream)
```

`STORAGE_IMPL` 是抽象存储接口，生产环境通常为 MinIO 实现。空文件返回错误而非 200+空响应。

**步骤 4: MIME 类型判断与响应**

```python
return await send_file(
    file,
    as_attachment=True,
    attachment_filename=doc[0].name,
    mimetype=_mimetype_for_document(doc[0]),
)
```

`_mimetype_for_document` 从文件名后缀推断 MIME 类型：

```python
def _mimetype_for_document(doc) -> str:
    match = re.search(r"\.([^.]+)$", (doc.name or "").lower())
    if not match:
        return "application/octet-stream"
    ext = match.group(1)
    fallback_prefix = "image" if doc.type == FileType.VISUAL.value else "application"
    return CONTENT_TYPE_MAP.get(ext, f"{fallback_prefix}/{ext}")
```

- 已知扩展名（如 `pdf`/`docx`/`png`）使用预定义映射
- 未知扩展名根据 `doc.type` 推断前缀（`image/*` 或 `application/*`）
- 无扩展名回退到 `application/octet-stream`

**响应头关键点**:
- `Content-Disposition: attachment; filename="..."` — 强制浏览器下载而非预览
- `attachment_filename` 使用文档原始名称，保留扩展名和字符（Quart 自动处理 UTF-8 编码）

---

## 流程 2: 文档预览（内联渲染）

**端点**: `GET /documents/{doc_id}/preview`

与下载端点的核心区别是 **`Content-Disposition` 策略**：默认 `inline`，仅对高风险 MIME 类型（HTML/SVG）强制 `attachment`。

### 执行步骤

**步骤 1-3**: 权限校验、存储地址解析、文件读取（同流程 1）

**步骤 4: 扩展名与 MIME 推断**

```python
ext = re.search(r"\.([^.]+)$", doc.name.lower())
ext = ext.group(1) if ext else None
content_type = None
if ext:
    fallback_prefix = "image" if doc.type == FileType.VISUAL.value else "application"
    content_type = CONTENT_TYPE_MAP.get(ext, f"{fallback_prefix}/{ext}")
```

**步骤 5: 安全响应头应用**

调用 `apply_preview_file_response_headers` 根据文件类型决定内联或强制下载：

```python
apply_preview_file_response_headers(response, content_type, ext, doc.name)
```

`apply_preview_file_response_headers` 的安全逻辑（`file_response.py`）：

```python
def apply_preview_file_response_headers(response, content_type, ext, filename):
    if content_type:
        response.headers.set("Content-Type", content_type)
    if should_force_attachment(ext, content_type):
        response.headers.set("X-Content-Type-Options", "nosniff")
        response.headers.set("Content-Disposition", format_content_disposition("attachment", filename))
        return response
    # 否则允许内联预览
    response.headers.set("Content-Disposition", format_content_disposition("inline", filename))
```

**强制附件触发条件** (`should_force_attachment`):
- 扩展名在黑名单：`html`, `htm`, `svg`, `xml`, `xhtml`, `shtml`, `mhtml`
- Content-Type 在黑名单：`text/html`, `image/svg+xml`, `application/xml` 等

**安全动机**: HTML 和 SVG 文件可能包含恶意脚本。强制下载避免浏览器执行其中的 JavaScript，防止存储型 XSS 攻击。

**文件名编码处理** (`format_content_disposition`):

```python
def format_content_disposition(disposition: str, filename: str | None) -> str:
    base = str(filename).split("/")[-1].split("\\")[-1]
    ascii_fallback = ascii_content_disposition_filename(base) or "file"
    encoded = quote(base, safe="")
    return f"{disposition}; filename=\"{ascii_fallback}\"; filename*=UTF-8''{encoded}"
```

同时输出两个 filename 参数（RFC 6266）：
- `filename="..."` — ASCII 安全回退，非 ASCII 字符替换为 `_`
- `filename*=UTF-8''...` — URL 编码的完整 UTF-8 名称，现代浏览器优先使用

路径分隔符被剥离，防止路径遍历注入到响应头。

---

## 流程 3: 缩略图批量获取

**端点**: `GET /thumbnails?doc_ids=id1&doc_ids=id2`

前端文档列表用于批量拉取缩略图，避免逐个请求。

### 执行步骤

```python
doc_ids = request.args.getlist("doc_ids")
if not doc_ids:
    return get_json_result(data=False, message='Lack of "Document ID"', code=RetCode.ARGUMENT_ERROR)

validate_rest_api_ids(doc_ids, "doc_ids")  # 校验非空、无重复

docs = DocumentService.get_thumbnails(doc_ids)
for doc_item in docs:
    if doc_item["thumbnail"] and not doc_item["thumbnail"].startswith(IMG_BASE64_PREFIX):
        doc_item["thumbnail"] = f"/api/v1/documents/images/{doc_item['kb_id']}-{doc_item['thumbnail']}"

return get_json_result(data={d["id"]: d["thumbnail"] for d in docs})
```

### 缩略图两种存储形态

| 形态 | 判断依据 | 返回值 | 适用场景 |
|------|---------|--------|---------|
| Base64 内联 | 以 `IMG_BASE64_PREFIX` 开头 | 原样返回 data URI | 小尺寸缩略图，减少请求数 |
| 存储对象引用 | 其他值 | 重写为 `/api/v1/documents/images/{kb_id}-{key}` | 大尺寸缩略图，走 CDN 缓存 |

**响应格式**:
```json
{
  "doc_id_1": "data:image/png;base64,iVBORw0KG...",
  "doc_id_2": "/api/v1/documents/images/kb123abc-thumbnail_xyz.png"
}
```

**权限注意**: 该端点**未做知识库权限校验**——只要提供有效 `doc_ids` 即返回缩略图 URL。虽然实际图片下载时 `/documents/images/` 端点会做校验，但缩略图 URL 本身泄漏了 `kb_id` 与对象 key。

---

## 流程 4: 图片服务

**端点**: `GET /documents/images/{image_id}`

`image_id` 是复合 ID，格式为 `{dataset_id}-{object_key}`。

### 执行步骤

**步骤 1: 复合 ID 解析**

```python
def _parse_document_image_id(image_id: str) -> tuple[str, str] | None:
    parts = image_id.split("-", 1)
    if len(parts) != 2 or not re.fullmatch(r"[0-9a-fA-F]{32}", parts[0]) or not parts[1]:
        return None
    return parts[0], parts[1]
```

**关键约束**:
- **仅在第一个连字符处分割**（`split("-", 1)`），因为对象 key 本身可能含连字符（如 `page-1.png`）
- `dataset_id` 必须是 32 位十六进制字符串（UUID hex 格式），格式不符直接拒绝
- 对象 key 不能为空

**步骤 2: 存储读取**

```python
bkt, nm = parsed
data = await thread_pool_exec(settings.STORAGE_IMPL.get, bkt, nm)
if data is None:
    logging.warning("get_document_image: storage miss image_id: %s, bucket: %s, key: %s", image_id, bkt, nm)
    return get_data_error_result(message="Image not found.")
```

存储读取通过 `thread_pool_exec` 卸载到线程池，避免阻塞事件循环（Quart 是异步框架，MinIO SDK 是同步的）。

**步骤 3: Content-Type 三级判定**

图片端点不信任单一来源，按优先级依次尝试：

```python
def _content_type_for_document_image(object_name: str, data: bytes) -> str:
    ext_match = re.search(r"\.([^.]+)$", object_name.lower())
    if ext_match:
        content_type = CONTENT_TYPE_MAP.get(ext_match.group(1))
        if content_type and content_type.startswith("image/"):
            return content_type
    detected = _detect_image_content_type_from_bytes(data)
    if detected:
        return detected
    return "application/octet-stream"
```

| 优先级 | 判定方式 | 约束 |
|--------|---------|------|
| 1 | 扩展名查 `CONTENT_TYPE_MAP` | 结果必须以 `image/` 开头，否则视为无效继续下一级 |
| 2 | 文件头魔术字节嗅探 | 不依赖文件名，抗扩展名伪造 |
| 3 | `application/octet-stream` | 无法识别时的安全兜底（浏览器不会渲染） |

**魔术字节检测实现**:

```python
def _detect_image_content_type_from_bytes(data: bytes) -> str | None:
    if data.startswith(b"\x89PNG\r\n\x1a\n"):
        return "image/png"
    if data[:3] == b"\xff\xd8\xff":
        return "image/jpeg"
    if data[:6] in (b"GIF87a", b"GIF89a"):
        return "image/gif"
    if len(data) >= 12 and data[:4] == b"RIFF" and data[8:12] == b"WEBP":
        return "image/webp"
    if data[:2] == b"BM":
        return "image/bmp"
    return None
```

**设计意图**: 解析阶段产出的图片（PDF 页面截图、表格切图）对象 key 可能不带扩展名，魔术字节兜底保证浏览器能正确渲染；同时第一级要求 `image/*` 前缀，避免 `.html` 这类扩展名被当成图片内联返回。

---

## 流程 5: 沙盒工件获取

**端点**: `GET /documents/artifact/{filename}`

代码执行沙盒（Agent 的 Code 组件）运行产物的下载通道，例如生成的图表 PNG、导出的 CSV。这类文件不属于任何知识库，因此授权逻辑与文档完全不同。

### 执行步骤

**步骤 1: 文件名三重校验**

```python
bucket = SANDBOX_ARTIFACT_BUCKET
basename = os.path.basename(filename)
if basename != filename or "/" in filename or "\\" in filename:
    return get_data_error_result(message="Invalid filename.")
ext = os.path.splitext(basename)[1].lower()
if ext not in ARTIFACT_CONTENT_TYPES:
    return get_data_error_result(message="invalid file type")
```

- `basename != filename` — 拒绝任何带目录成分的路径（含 `../`）
- 显式拒绝两种路径分隔符（跨平台）
- 扩展名必须在白名单内

**扩展名白名单**:

```python
ARTIFACT_CONTENT_TYPES = {
    ".png": "image/png",
    ".jpg": "image/jpeg",
    ".jpeg": "image/jpeg",
    ".svg": "image/svg+xml",
    ".pdf": "application/pdf",
    ".csv": "text/csv",
    ".json": "application/json",
    ".html": "text/html",
}
```

与文档端点的黑名单策略相反，工件端点用白名单——沙盒是不可信执行环境，只允许已知安全的输出格式。

**步骤 2: 双通道授权**

工件没有 `kb_id` 之类的归属字段，因此授权通过"这个文件是否出现在你能访问的会话里"来反推：

```python
session_id = request.args.get("session_id", "")
if not await thread_pool_exec(_sandbox_artifact_accessible, basename, current_user.id) \
   and not await thread_pool_exec(_sandbox_artifact_session_accessible, session_id, current_user.id):
    return get_data_error_result(message="Artifact not found.")
```

**通道 A：文件名反查会话**

```python
@DB.connection_context()
def _sandbox_artifact_dialog_ids_for_user(filename: str, user_id: str) -> list[str]:
    artifact_ref = f"documents/artifact/{filename}"
    rows = (
        API4Conversation.select(API4Conversation.dialog_id)
        .where(
            ((API4Conversation.user_id == user_id) | (API4Conversation.exp_user_id == user_id)),
            (API4Conversation.message.contains(filename)
             | API4Conversation.message.contains(artifact_ref)),
        )
        .distinct()
    )
    return [row.dialog_id for row in rows if row.dialog_id]

@DB.connection_context()
def _sandbox_artifact_accessible(filename: str, user_id: str) -> bool:
    for dialog_id in _sandbox_artifact_dialog_ids_for_user(filename, user_id):
        if UserCanvasService.accessible(dialog_id, user_id):
            return True
    return False
```

**通道 B：显式 session_id**

```python
@DB.connection_context()
def _sandbox_artifact_session_accessible(session_id: str, user_id: str) -> bool:
    if not session_id:
        return False
    conv = API4Conversation.get_or_none(API4Conversation.id == session_id)
    if not conv:
        return False
    if str(conv.user_id) != str(user_id) and str(conv.exp_user_id or "") != str(user_id):
        return False
    return UserCanvasService.accessible(conv.dialog_id, user_id)
```

- **通道 A** — 扫描用户所有会话，看 `message` JSON 字段是否包含该文件名或其引用路径，再检查对应 `dialog_id` 的画布权限
- **通道 B** — 前端传 `session_id` 参数时，直接校验该会话归属与画布权限
- 只要任一通道通过即放行，两者均失败时统一返回 `"Artifact not found."`

**步骤 3: 存储读取与安全响应头**

```python
data = await thread_pool_exec(settings.STORAGE_IMPL.get, bucket, basename)
content_type = ARTIFACT_CONTENT_TYPES.get(ext, "application/octet-stream")

response = await make_response(data)
safe_filename = re.sub(r"[^\w.\-]", "_", basename)
apply_safe_file_response_headers(response, content_type, ext)
if not response.headers.get("Content-Disposition"):
    response.headers.set("Content-Disposition", f'inline; filename="{safe_filename}"')

return response
```

- `thread_pool_exec` 将 MinIO 同步调用移出 Quart 事件循环
- `safe_filename` — 清理非字母数字字符，防止响应头注入（虽然已经通过 basename 检查，保留是纵深防御）
- `apply_safe_file_response_headers` — 为 `.html` / `.svg` 强制 `attachment` + `nosniff`
- 如果未被覆盖，默认 `inline` 方便浏览器直接渲染

---

## 设计考量与安全说明

### 1. 统一错误消息防枚举

下载、预览、工件三类端点都把"无权限"和"不存在"合并为同一条错误消息。这样攻击者无法通过遍历 ID 判断资源是否存在，代价是排障时不易区分两种失败原因（需依赖服务端日志）。

### 2. 黑名单 vs 白名单的分层选择

| 场景 | 策略 | 原因 |
|------|------|------|
| 文档预览 | 黑名单（`FORCE_ATTACHMENT_EXTENSIONS`） | 文档类型开放，需支持任意用户上传格式的预览 |
| 沙盒工件 | 白名单（`ARTIFACT_CONTENT_TYPES`） | 沙盒输出格式可枚举，且执行环境不可信 |
| 图片服务 | 白名单（要求 `image/*`） | 该端点语义上只应返回图片 |

### 3. list_thumbnails 的权限缺口

`GET /thumbnails` 只校验 `doc_ids` 参数格式，不校验调用者是否有权访问这些文档所在知识库。返回值中包含 `kb_id` 与缩略图对象 key，属于信息泄漏。实际图片字节仍受 `/documents/images/` 端点保护，但拼出的 URL 暴露了内部存储结构。

### 4. 工件授权的性能隐患

通道 A 使用 `API4Conversation.message.contains(filename)` 做全表子串匹配，`message` 是存储完整对话历史的大字段。会话量增长后这条查询会成为瓶颈，且无法利用索引。前端应始终携带 `session_id` 走通道 B。

---

## 性能特征

| 端点 | DB 查询 | 存储读取 | 异步卸载 | 主要开销 |
|------|---------|---------|---------|---------|
| 下载 | 2-3 次（权限+文档+File2Document） | 1 次全量读入 `BytesIO` | 否 | 大文件全量载入内存 |
| 预览 | 2-3 次 | 1 次全量读入 | 否 | 同上 |
| 缩略图批量 | 1 次（`IN` 查询） | 0 次 | 否 | 仅返回 URL，开销最小 |
| 图片服务 | 0 次 | 1 次 | 是（`thread_pool_exec`） | 存储 RTT |
| 沙盒工件 | 1-2 次（通道 A 为全表 LIKE） | 1 次 | 是 | 通道 A 的子串扫描 |

**共同问题**: 所有端点都把文件完整读入内存后再响应，没有流式传输。大文件（如百 MB 级 PDF）下载会造成显著内存峰值。

---

## 相关流程

- 上传时的存储写入与 `File2Document` 关联建立：[upload-flow.md](./upload-flow.md)
- 删除时的存储对象清理：[delete-flow.md](./delete-flow.md)
- 缩略图的生成时机（解析阶段产出）：[parse-control-flow.md](./parse-control-flow.md)
- 文档列表查询（缩略图 URL 的消费方）：[query-flow.md](./query-flow.md)

---

## 代码追溯

```
GET /datasets/{id}/documents/{doc_id}
  └─ document_api.py:download (2140-2199)
       ├─ KnowledgebaseService.accessible
       ├─ DocumentService.accessible
       ├─ File2DocumentService.get_storage_address  (file2document_service.py:60-96)
       ├─ settings.STORAGE_IMPL.get
       └─ _mimetype_for_document (2131-2137) → CONTENT_TYPE_MAP

GET /documents/{doc_id}/preview
  └─ document_api.py:get (2095-2128)
       └─ apply_preview_file_response_headers  (file_response.py)
            ├─ should_force_attachment
            └─ format_content_disposition → ascii_content_disposition_filename

GET /thumbnails
  └─ document_api.py:list_thumbnails (1294-1334)
       └─ DocumentService.get_thumbnails

GET /documents/images/{image_id}
  └─ document_api.py:get_document_image (1831-1869)
       ├─ _parse_document_image_id (1786-1802)
       └─ _content_type_for_document_image (1819-1828)
            └─ _detect_image_content_type_from_bytes (1805-1816)

GET /documents/artifact/{filename}
  └─ document_api.py:get_artifact (1921-1971)
       ├─ _sandbox_artifact_accessible (1901-1906)
       │    └─ _sandbox_artifact_dialog_ids_for_user (1884-1898)
       ├─ _sandbox_artifact_session_accessible (1909-1918)
       └─ apply_safe_file_response_headers
```

**关键文件**:
- `api/apps/restful_apis/document_api.py` — 全部端点实现
- `api/utils/file_response.py` — `CONTENT_TYPE_MAP`、强制附件判定、RFC 6266 文件名编码
- `api/db/services/file2document_service.py` — 存储地址解析
- `api/db/services/document_service.py` — `get_thumbnails`、`accessible`
