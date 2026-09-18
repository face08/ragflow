# 文档查询流程

## 概述

文档查询流程负责在知识库中检索、过滤和聚合文档信息，支持分页、排序、关键词搜索、元数据过滤和统计聚合。查询端点设计为单一入口，通过 `type` 参数区分列表查询与统计聚合。

**核心端点**:
- `GET /datasets/{id}/documents` — 列表查询（默认）
- `GET /datasets/{id}/documents?type=filter` — 统计聚合查询

**架构特征**:
- **两阶段过滤**: 元数据 Push-down（向量库）→ 内存过滤（文档属性 + 时间范围）
- **元数据回退机制**: Push-down 失败时内存完整扫描，保证查询鲁棒性
- **分页即时获取元数据**: 仅为当前页文档加载元数据，避免全库扫描
- **聚合统计与空元数据检测**: 单次查询同时返回 suffix/run_status/metadata 三维统计

---

## 流程 1: 列表查询（List Documents）

**端点**: `GET /datasets/{id}/documents`

### 请求参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `page` | int | 1 | 页码（负数或非法值回退到 1） |
| `page_size` | int | 30 | 每页文档数（负数或非法值回退到 30） |
| `orderby` | string | `create_time` | 排序字段（支持 `create_time`/`update_time`/`name`） |
| `desc` | bool | true | 降序排序 |
| `keywords` | string | "" | 文档名关键词（不区分大小写） |
| `suffix` | array | [] | 文件后缀过滤（如 `["pdf", "docx"]`） |
| `types` | array | [] | 文档类型过滤（如 `["pdf", "folder"]`） |
| `run` | array | [] | 解析状态过滤（支持数字 `["0","1"]` 或文本 `["UNSTART","RUNNING"]`） |
| `create_time_from` | int | 0 | 创建时间下界（Unix 时间戳，0 表示无限制） |
| `create_time_to` | int | 0 | 创建时间上界（Unix 时间戳，0 表示无限制） |
| `metadata` | JSON string | {} | 简单元数据过滤（精确匹配，同 key 内 OR、跨 key 间 AND） |
| `metadata_condition` | JSON string | {} | 复杂元数据条件（支持 `=`/`!=`/`>`/`contains`/`in` 等操作符） |
| `return_empty_metadata` | bool | false | 仅返回元数据为空的文档 |
| `id` | string | null | 单文档 ID 查询（提供时忽略其他过滤） |
| `ids` | array | [] | 多文档 ID 批量查询 |
| `name` | string | null | 精确文档名查询 |

### 执行步骤

**步骤 1: 权限校验**
```python
if not KnowledgebaseService.accessible(kb_id=dataset_id, user_id=tenant_id):
    return get_error_data_result(message=f"You don't own the dataset {dataset_id}.")
```

**步骤 2: 参数验证与归一化**

分页参数容错处理，避免非法值泄漏内部 SQL 错误：

```python
page = validate_rest_api_page(q.get("page", DEFAULT_PAGE))
page_size = validate_rest_api_page_size(q.get("page_size", DEFAULT_PAGE_SIZE))

orderby = q.get("orderby", "create_time")
if orderby not in ("create_time", "update_time", "name"):
    return RetCode.ARGUMENT_ERROR, f"invalid orderby field: {orderby}", [], 0
desc = str(q.get("desc", "true")).strip().lower() != "false"
```

**步骤 3: 解析状态过滤归一化**

同时支持数字与文本两种状态表示，通过 `TaskStatus` 枚举名映射：

```python
def _parse_run_status_filter(req_args):
    raw_statuses = _get_query_values(req_args, "run", "run_status")
    status_text_to_numeric = {status.name: status.value for status in TaskStatus}
    valid_statuses = set(status_text_to_numeric.values())
    converted = [status_text_to_numeric.get(status.upper(), status) for status in raw_statuses]
    invalid_statuses = {status for status in converted if status not in valid_statuses}
    return converted, invalid_statuses
```

`_get_query_values` 同时兼容 `run=0&run=1` 与 `run[]=0&run[]=1` 两种数组风格，并剔除空白值：

```python
def _get_query_values(req_args, *names):
    values = []
    for name in names:
        values.extend(req_args.getlist(name))
        values.extend(req_args.getlist(f"{name}[]"))
    return [str(value).strip() for value in values if value is not None and str(value).strip()]
```

**步骤 4: 元数据过滤解析（Push-down 前置）**

`_parse_doc_id_filter_with_metadata` 将元数据条件转为 `doc_ids_filter` 白名单：

```python
metas = dict()
if metadata_condition or metadata:
    metas = DocMetadataService.get_flatted_meta_by_kbs([kb_id])

doc_ids_filter = None
if metadata_condition:
    doc_ids_filter = set(meta_filter(metas, convert_conditions(metadata_condition),
                                     metadata_condition.get("logic", "and")))
    if metadata_condition.get("conditions") and not doc_ids_filter:
        return RetCode.SUCCESS, "", [], return_empty_metadata
```

`metadata` 简单过滤的集合运算语义（同 key OR，跨 key AND）：

```python
for key, values in metadata.items():
    key_doc_ids = set()
    for value in values:
        key_doc_ids.update(metas.get(key, {}).get(value, []))
    if metadata_doc_ids is None:
        metadata_doc_ids = key_doc_ids
    else:
        metadata_doc_ids &= key_doc_ids
    if not metadata_doc_ids:
        return RetCode.SUCCESS, "", [], return_empty_metadata
```

**特殊键 `empty_metadata`**: 出现在 `metadata` 中时会翻转为 `return_empty_metadata=True`，并清空其余过滤条件：

```python
if isinstance(metadata, dict) and metadata.get("empty_metadata"):
    return_empty_metadata = True
    metadata = {k: v for k, v in metadata.items() if k != "empty_metadata"}
if return_empty_metadata:
    metadata_condition = {}
    metadata = {}
```

**步骤 5: 单文档/批量 ID 优先级处理**

`id` 参数优先级最高，忽略其他过滤；`ids` 批量查询需校验 ID 合法性（非空、无重复）：

```python
doc_id = q.get("id")
if doc_id:
    if not DocumentService.query(id=doc_id, kb_id=dataset_id):
        return RetCode.DATA_ERROR, f"you don't own the document {doc_id}", [], 0
    doc_ids_filter = [doc_id]

doc_ids = q.getlist("ids")
validate_rest_api_ids(doc_ids)  # 校验无空串、无重复
if doc_id and len(doc_ids) > 0:
    return RetCode.DATA_ERROR, f"Should not provide both 'id':{doc_id} and 'ids'{doc_ids}", [], 0
if len(doc_ids) > 0:
    doc_ids_filter = doc_ids
```

**步骤 6: MySQL 查询与 JOIN**

`DocumentService.get_by_kb_id` 通过三表 JOIN 获取基础字段和管道名称：

```python
docs = (
    Document.select(*fields, UserCanvas.title.alias("pipeline_name"), User.nickname)
    .join(File2Document, on=(File2Document.document_id == Document.id))
    .join(UserCanvas, on=(Document.pipeline_id == UserCanvas.id), join_type=JOIN.LEFT_OUTER)
    .join(File, on=(File.id == File2Document.file_id))
    .join(User, on=(Document.created_by == User.id), join_type=JOIN.LEFT_OUTER)
    .where(Document.kb_id == kb_id)
)
```

应用过滤条件（WHERE 子句）：

```python
if keywords:
    docs = docs.where(fn.LOWER(Document.name).contains(keywords.lower()))
if doc_ids is not None:
    docs = docs.where(Document.id.in_(doc_ids))
if run_status:
    docs = docs.where(Document.run.in_(run_status))
if types:
    docs = docs.where(Document.type.in_(types))
if suffix:
    docs = docs.where(Document.suffix.in_(suffix))
if name:
    docs = docs.where(Document.name == name)
```

**空元数据反向过滤**: `return_empty_metadata=True` 时排除已有元数据的文档：

```python
if return_empty_metadata:
    metadata_map = DocMetadataService.get_metadata_for_documents(None, kb_id)
    doc_ids_with_metadata = set(metadata_map.keys())
    if doc_ids_with_metadata:
        docs = docs.where(Document.id.not_in(doc_ids_with_metadata))
```

分页与计数（`count()` 先于 `paginate()` 避免影响总数）：

```python
count = docs.count()
if desc:
    docs = docs.order_by(Document.getter_by(orderby).desc())
else:
    docs = docs.order_by(Document.getter_by(orderby).asc())

if page_number and items_per_page:
    docs = docs.paginate(page_number, items_per_page)
```

**步骤 7: 元数据即时注入（仅当前页）**

查询完成后为当前页文档批量加载元数据，避免全库扫描：

```python
docs_list = list(docs.dicts())
if return_empty_metadata:
    for doc in docs_list:
        doc["meta_fields"] = {}
else:
    doc_ids_on_page = [doc["id"] for doc in docs_list]
    metadata_map = DocMetadataService.get_metadata_for_documents(doc_ids_on_page, kb_id)
    for doc in docs_list:
        doc["meta_fields"] = metadata_map.get(doc["id"], {})
```

**步骤 8: 时间范围内存过滤**

时间过滤在 MySQL 查询之后于内存执行，**这是已知的设计缺陷**：

```python
create_time_from = int(q.get("create_time_from", 0))
create_time_to = int(q.get("create_time_to", 0))
if create_time_from or create_time_to:
    docs = [d for d in docs
            if (create_time_from == 0 or d.get("create_time", 0) >= create_time_from)
            and (create_time_to == 0 or d.get("create_time", 0) <= create_time_to)]
```

**问题**: 时间过滤发生在分页之后，导致返回的 `docs` 数量可能少于 `page_size`，且 `total` 仍是未过滤的总数。分页与时间过滤组合使用时结果不准确。正确做法应将时间条件下推到 SQL WHERE 子句。

**步骤 9: 响应字段映射与缩略图 URL 重写**

```python
renamed_doc_list = [map_doc_keys(doc) for doc in payload]
for doc_item in renamed_doc_list:
    if doc_item["thumbnail"] and not doc_item["thumbnail"].startswith(IMG_BASE64_PREFIX):
        doc_item["thumbnail"] = f"/api/v1/documents/images/{dataset_id}-{doc_item['thumbnail']}"
    if doc_item.get("source_type"):
        doc_item["source_type"] = doc_item["source_type"].split("/")[0]
    if doc_item["parser_config"].get("metadata"):
        doc_item["parser_config"]["metadata"] = turn2jsonschema(doc_item["parser_config"]["metadata"])
return get_json_result(data={"total": total, "docs": renamed_doc_list})
```

三项转换：
- **缩略图**: 非 base64 内联的缩略图重写为图片下载 URL
- **source_type**: 截断为一级类型（`"connector/notion"` → `"connector"`）
- **metadata schema**: `parser_config.metadata` 转为 JSON Schema 格式返回给前端

---

## 流程 2: 统计聚合查询（Filter Aggregation）

**端点**: `GET /datasets/{id}/documents?type=filter`

用于前端过滤面板渲染，一次查询返回三个维度的计数分布。

### 响应格式

```json
{
  "total": 42,
  "filter": {
    "suffix": {"pdf": 20, "docx": 15, "txt": 7},
    "run_status": {"0": 10, "1": 2, "3": 28, "4": 2},
    "metadata": {
      "author": {"John": 5, "Jane": 3},
      "department": {"Sales": 8},
      "empty_metadata": {"true": 26}
    }
  }
}
```

`run_status` 的键为 `TaskStatus` 数值：`0`=UNSTART、`1`=RUNNING、`2`=CANCEL、`3`=DONE、`4`=FAIL、`5`=SCHEDULE。

### 执行步骤

**步骤 1: 参数复用**

聚合查询复用列表查询的 `keywords`/`run`/`types`/`suffix` 过滤条件，但**不接受分页参数**——统计必须覆盖全量匹配集合：

```python
def _get_doc_filters_with_request(req, dataset_id):
    if not KnowledgebaseService.accessible(kb_id=dataset_id, user_id=tenant_id):
        return RetCode.AUTHENTICATION_ERROR, f"You don't own the dataset {dataset_id}.", {}, 0
    keywords = req.args.get("keywords", "")
    run_status, invalid = _parse_run_status_filter(req.args)
    suffix = _get_query_values(req.args, "suffix")
    types = _get_query_values(req.args, "types")
```

**步骤 2: 全量扫描与三维计数**

`get_filter_by_kb_id` 只 SELECT 三个字段（`run`/`suffix`/`id`），减少 IO：

```python
rows = query.select(cls.model.run, cls.model.suffix, cls.model.id)
total = rows.count()

doc_ids = [row.id for row in rows]
metadata = {}
if doc_ids:
    try:
        metadata = DocMetadataService.get_metadata_for_documents(doc_ids, kb_id)
    except Exception as e:
        logging.warning(f"Failed to fetch metadata from ES/Infinity: {e}")
```

**关键设计**: 元数据获取包裹在 try/except 中。向量库不可用时聚合退化为仅返回 suffix/run_status 统计，而非整体失败。

**步骤 3: 元数据值展平计数**

列表类型元数据值会被展开逐一计数，空值（`None`/空白字符串）被跳过：

```python
for row in rows:
    suffix_counter[row.suffix] = suffix_counter.get(row.suffix, 0) + 1
    run_status_counter[str(row.run)] = run_status_counter.get(str(row.run), 0) + 1
    meta_fields = metadata.get(row.id, {})
    if not meta_fields:
        empty_metadata_count += 1
        continue
    has_valid_meta = False
    for key, value in meta_fields.items():
        values = value if isinstance(value, list) else [value]
        for vv in values:
            if vv is None:
                continue
            if isinstance(vv, str) and not vv.strip():
                continue
            metadata_counter.setdefault(key, {})
            sv = str(vv)
            metadata_counter[key][sv] = metadata_counter[key].get(sv, 0) + 1
            has_valid_meta = True
    if not has_valid_meta:
        empty_metadata_count += 1
```

**步骤 4: 空元数据计数注入**

`empty_metadata` 作为伪 key 混入 `metadata` 计数字典，供前端渲染"无元数据"过滤项：

```python
metadata_counter["empty_metadata"] = {"true": empty_metadata_count}
```

**两类空元数据**:
1. 文档在向量库中无元数据记录（`meta_fields` 为空 dict）
2. 文档有元数据 key，但所有值均为 `None` 或空白字符串（`has_valid_meta=False`）

两者都计入 `empty_metadata`，与查询侧 `return_empty_metadata` 的语义（向量库无记录）**并不完全对齐**——第二类文档在聚合中显示为空，但在 `return_empty_metadata=true` 查询中不会返回。

---

## 设计考量与已知问题

### 1. 元数据过滤的双通道设计

系统提供两套元数据过滤入口，语义不同：

| 入口 | 数据源 | 语义 | 适用场景 |
|------|--------|------|---------|
| `metadata` | `get_flatted_meta_by_kbs` 扁平索引 | 精确值匹配，同 key OR / 跨 key AND | 前端过滤面板点选 |
| `metadata_condition` | 同上 + `meta_filter` 条件求值 | 操作符表达式，`logic` 控制 and/or | 高级搜索、API 集成 |

两者都在 SQL 查询**之前**求值为 `doc_ids` 白名单，再作为 `Document.id.in_(...)` 下推。当匹配集合为空时提前返回，避免无意义的 SQL 查询。

**规模隐患**: `get_flatted_meta_by_kbs` 加载整个知识库的元数据倒排索引。文档量大（万级以上）且元数据字段多时，该调用会成为查询延迟主要来源。`doc_ids` 白名单也可能膨胀到导致 `IN (...)` 子句超出 MySQL 限制。

### 2. 时间过滤的分页错位

如步骤 8 所述，`create_time_from`/`create_time_to` 在分页之后于内存过滤：

```
SQL: WHERE kb_id=? LIMIT 30 OFFSET 0   → 30 条
内存: filter(create_time in range)      → 可能只剩 5 条
响应: {"total": 已统计的全量数, "docs": [5 条]}
```

结果是前端分页器显示的总页数与实际可翻页数不一致，且每页文档数不定。规避方式是不同时使用时间过滤与分页，或在客户端做二次聚合。

### 3. 聚合查询与列表查询的过滤能力不对称

`type=filter` 只支持 `keywords`/`run`/`types`/`suffix`，**不支持** `metadata`/`metadata_condition`/`create_time_*`/`ids`。因此在已应用元数据过滤的前端视图中，过滤面板的计数仍是元数据过滤前的全量分布。

### 4. 元数据加载策略对比

| 场景 | 加载范围 | 调用 |
|------|---------|------|
| 列表查询（普通） | 仅当前页 doc_ids | `get_metadata_for_documents(page_ids, kb_id)` |
| 列表查询（空元数据过滤） | 全库 doc_ids | `get_metadata_for_documents(None, kb_id)` |
| 元数据条件过滤 | 全库倒排索引 | `get_flatted_meta_by_kbs([kb_id])` |
| 聚合查询 | 全部匹配 doc_ids | `get_metadata_for_documents(doc_ids, kb_id)` |

只有第一种是 O(page_size)，其余均为 O(库内文档数)。

---

## 性能特征

| 操作 | MySQL 查询 | 向量库查询 | 内存操作 |
|------|-----------|-----------|---------|
| 基础列表查询 | 1 次 count + 1 次 select（4 表 JOIN） | 1 次（当前页元数据） | 字段映射 |
| 关键词/后缀/状态过滤 | 同上（WHERE 下推） | 同上 | 同上 |
| 元数据过滤 | 同上 + `IN` 白名单 | 1 次全库倒排 + 1 次当前页 | 集合交并运算 |
| 空元数据过滤 | 同上 + `NOT IN` 全库 ID | 1 次全库元数据 | - |
| 时间范围过滤 | 同上 | 同上 | 分页后线性过滤 |
| 统计聚合 | 1 次 count + 1 次全表 select | 1 次全量元数据 | 三维计数遍历 |

---

## 相关流程

- [upload-flow.md](./upload-flow.md) — 文档创建后即可被查询
- [metadata-flow.md](./metadata-flow.md) — 元数据写入与倒排索引构建
- [parse-control-flow.md](./parse-control-flow.md) — `run` 字段的状态迁移来源
- [update-flow.md](./update-flow.md) — 重命名影响 `keywords`/`name` 查询结果

---

## 代码追溯

```
GET /datasets/{id}/documents
  └─ list_docs (api/apps/restful_apis/document_api.py)
      ├─ type=filter?
      │   └─ _get_doc_filters_with_request
      │       ├─ KnowledgebaseService.accessible
      │       ├─ _parse_run_status_filter / _get_query_values
      │       └─ DocumentService.get_filter_by_kb_id
      │           └─ DocMetadataService.get_metadata_for_documents
      └─ 默认列表
          └─ _get_docs_with_request
              ├─ KnowledgebaseService.accessible
              ├─ validate_rest_api_page / validate_rest_api_page_size
              ├─ _parse_run_status_filter
              ├─ _parse_doc_id_filter_with_metadata
              │   ├─ DocMetadataService.get_flatted_meta_by_kbs
              │   └─ meta_filter / convert_conditions
              ├─ validate_rest_api_ids
              ├─ DocumentService.get_by_kb_id
              │   ├─ Peewee JOIN (File2Document / File / UserCanvas / User)
              │   └─ DocMetadataService.get_metadata_for_documents
              └─ 时间范围内存过滤
          └─ map_doc_keys + 缩略图 URL 重写 + turn2jsonschema
```

**关键文件**:
- `api/apps/restful_apis/document_api.py` — 查询端点与参数解析（`list_docs`、`_get_docs_with_request`、`_get_doc_filters_with_request`、`_parse_doc_id_filter_with_metadata`）
- `api/db/services/document_service.py` — 数据层（`get_by_kb_id`、`get_filter_by_kb_id`）
- `api/db/services/doc_metadata_service.py` — 元数据读取（`get_metadata_for_documents`、`get_flatted_meta_by_kbs`）
