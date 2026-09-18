# 文件-文档关联流程

> **范围**: File ↔ Document N:M 映射管理，文件链接到知识库

## 概述

File2Document 中间表是双视图架构的核心桥梁，使同一物理文件可被多个知识库以不同配置使用。本文档覆盖关联的创建、替换、查询逻辑。

---

## 流程 1: 文件链接到知识库

**端点**: `POST /files/link-to-datasets`

**用途**: 将已存在的文件（可能来自文件系统视图上传）批量关联到一个或多个知识库，生成对应的 Document 记录。

### 步骤 1: 参数解析与模式识别

```python
# file2document_api.py:101-107
req = await get_request_json()
kb_ids = req["kb_ids"]      # 目标知识库 ID 列表
file_ids = req["file_ids"]  # 源文件 ID 列表
mode = (request.args.get("mode", "replace") or "replace").lower()
if mode not in {"replace", "add"}:
    return get_json_result(code=RetCode.ARGUMENT_ERROR, message="mode must be 'add' or 'replace'")
```

**mode 语义**:
- `add`: 仅添加缺失的关联，保留现有链接（文件可同时属于多个知识库）
- `replace`: 删除不在 `kb_ids` 中的现有关联，添加缺失关联（文件从旧知识库转移）

### 步骤 2: 前置验证

```python
# file2document_api.py:110-138
files = FileService.get_by_ids(file_ids)
files_set = {file.id: file for file in files}

# 验证所有文件存在
for file_id in file_ids:
    if not files_set.get(file_id):
        return get_data_error_result(message="File not found!")

# 验证所有知识库存在
kb_map = {}
for kb_id in kb_ids:
    e, kb = KnowledgebaseService.get_by_id(kb_id)
    if not e:
        return get_data_error_result(message="Can't find this dataset!")
    kb_map[kb_id] = kb
```

**校验点**:
1. 所有 `file_ids` 必须存在
2. 所有 `kb_ids` 必须存在
3. 提前失败策略：任何资源缺失都立即终止，不启动后台任务

### 步骤 3: 文件夹展开

```python
# file2document_api.py:140-147
all_file_ids = []
for file_id in file_ids:
    file = files_set[file_id]
    if file.type == FileType.FOLDER.value:
        all_file_ids.extend(FileService.get_all_innermost_file_ids(file_id, []))
    else:
        all_file_ids.append(file_id)
```

**递归处理**: 如果请求中包含文件夹，递归获取所有叶子文件（`get_all_innermost_file_ids`），最终操作的是实际文件而非文件夹节点。

### 步骤 4: 权限校验

```python
# file2document_api.py:150-180
for file_id in all_file_ids:
    e, file = FileService.get_by_id(file_id)
    if not e or not file:
        return get_data_error_result(message="File not found!")
    if not check_file_team_permission(file, user_id):
        return get_data_error_result(message="no authorization")

for kb_id, kb in kb_map.items():
    if not check_kb_team_permission(kb, user_id):
        return get_data_error_result(message="no authorization")
```

**双重授权**:
1. 每个展开后的文件必须通过团队权限检查
2. 每个目标知识库必须通过团队权限检查

### 步骤 5: 后台异步处理

```python
# file2document_api.py:182-194
loop = asyncio.get_running_loop()
future = loop.run_in_executor(None, _convert_files, all_file_ids, kb_ids, user_id, mode)
future.add_done_callback(lambda f: logging.error("_convert_files failed: %s", f.exception()) if f.exception() else None)
return get_json_result(data=True)
```

**异步卸载**: 对于包含大量文件的文件夹（可能展开成数百上千个文件），在线程池中执行批量关联，避免阻塞事件循环导致 504 超时。

### 步骤 6: 核心关联逻辑（`_convert_files`）

```python
# file2document_api.py:39-95
def _convert_files(file_ids, kb_ids, user_id, mode):
    replace_existing = mode == "replace"
    kb_ids = set(kb_ids)
    
    for id in file_ids:
        e, file = FileService.get_by_id(id)
        if not e:
            continue
        
        # 1. 查询现有关联
        existing_kb_ids = set()
        existing_links = File2DocumentService.get_by_file_id(id)
        for inform in existing_links:
            e, doc = DocumentService.get_by_id(inform.document_id)
            if e and doc:
                existing_kb_ids.add(doc.kb_id)
                
                # 2. 删除不在目标集中的现有关联（仅 replace 模式）
                if replace_existing and doc.kb_id not in kb_ids:
                    tenant_id = DocumentService.get_tenant_id(doc.id)
                    DocumentService.remove_document(doc, tenant_id)  # 删除 Document + Chunk
                    File2DocumentService.delete_by_document_id(doc.id)  # 删除关联记录
        
        # 3. 添加缺失的关联
        for kb_id in kb_ids:
            if kb_id in existing_kb_ids:
                continue  # 跳过已存在的关联
            
            e, kb = KnowledgebaseService.get_by_id(kb_id)
            if not e:
                continue
            
            # 创建 Document 记录
            filename = duplicate_name(DocumentService.query, name=file.name, kb_id=kb.id)
            doc = DocumentService.insert({
                "id": get_uuid(),
                "kb_id": kb.id,
                "parser_id": FileService.get_parser(file.type, filename, kb.parser_id),
                "pipeline_id": kb.pipeline_id,
                "parser_config": kb.parser_config,
                "created_by": user_id,
                "type": file.type,
                "name": filename,
                "suffix": Path(filename).suffix.lstrip("."),
                "location": file.location,  # 共享物理存储位置
                "size": file.size,
            })
            
            # 创建 File2Document 关联
            File2DocumentService.insert({
                "id": get_uuid(),
                "file_id": id,
                "document_id": doc.id,
            })
```

**关键行为**:
1. **replace 模式清理**: 删除旧关联时同步删除 Document（触发 Chunk 级联删除）和 File2Document 记录
2. **共享存储**: 新 Document 的 `location` 直接复用 File 的物理路径，避免重复上传
3. **解析器继承**: 从知识库继承 `parser_id`、`parser_config`、`pipeline_id`
4. **命名去重**: 使用 `duplicate_name` 确保同一知识库内文档名唯一

---

## 设计考量

### 1. 物理存储复用

同一文件链接到多个知识库时，**不会复制物理文件**：

- 所有关联的 Document 共享同一个 `location`（MinIO 对象路径）
- 删除某个知识库中的文档不会删除物理文件，除非所有关联都被移除
- 节省存储空间，特别是当文件被大量知识库引用时

### 2. 解析配置隔离

虽然共享物理文件，但每个知识库可用不同配置解析：

- Document A: `parser_id=resume`, `chunk_method=naive`
- Document B: `parser_id=paper`, `chunk_method=qa`
- 同一文件在不同知识库中产生不同的 Chunk 结果

### 3. 两种模式的使用场景

| 模式 | 适用场景 | 典型操作 |
|------|---------|---------|
| `add` | 文件需要被多个知识库共享 | 将公共文档库中的文件添加到新创建的专题知识库 |
| `replace` | 文件需要从一个知识库转移到另一个 | 将错误分类的文档从"技术文档"移动到"产品手册" |

### 4. 异步处理权衡

**优势**:
- 避免大批量操作时 HTTP 请求超时（Nginx 默认 60s）
- 提升用户体验（立即返回而非等待完成）

**劣势**:
- 客户端无法获知操作最终结果（成功/失败）
- 错误仅记录到服务端日志，无 API 响应

**改进方向**: 可引入任务队列（如 Celery）+ 任务状态查询端点，使客户端能轮询批量操作进度。

---

## 相关流程

- [上传流程](./upload-flow.md) — 上传时自动创建 File 和 Document，隐式建立关联
- [删除流程](./delete-flow.md) — 删除 Document 时处理 File2Document 关联
- [文件管理流程](./file-folder-flow.md) — File 的移动/重命名会级联更新关联的 Document

---

## 性能特征

| 操作 | 时间复杂度 | 数据库查询数 | 备注 |
|------|-----------|-------------|------|
| 链接 N 个文件到 M 个知识库 | O(N × M) | N × (2 + M) | 每个文件: 2 次查询现有关联 + M 次插入 |
| replace 模式额外开销 | O(K) | K × 3 | K = 被替换的旧关联数，每个需删除 Document + Chunk + File2Document |
| 文件夹展开 | O(D) | D | D = 文件夹深度，递归查询子节点 |

**优化建议**:
1. 批量查询现有关联可改用 `IN` 查询而非逐个 `get_by_file_id`
2. Document 批量插入可减少数据库往返

---

## 代码追溯

| 层次 | 文件 | 关键函数 |
|------|------|---------|
| API | `file2document_api.py` | `convert()` — 端点入口，验证与异步调度 |
| 业务逻辑 | `file2document_api.py` | `_convert_files()` — 同步批量关联逻辑 |
| 数据访问 | `file2document_service.py` | `get_by_file_id()`, `insert()`, `delete_by_document_id()` |
| 数据访问 | `file_service.py` | `get_all_innermost_file_ids()` — 文件夹递归展开 |
| 数据访问 | `document_service.py` | `remove_document()` — 删除 Document + Chunk |
