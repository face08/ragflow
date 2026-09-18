<!-- # 元数据流程 -->

## 概述

RAGFlow 的文档元数据系统采用 **向量索引独立存储** 架构，元数据完全存储在 ES/Infinity/OceanBase/GaussDB 中，MySQL 的 `meta_fields` 字段已废弃。这种设计支持灵活的元数据 schema、高性能过滤查询，以及跨租户隔离。

## 核心架构

### 索引命名规则
```python
# 元数据索引按租户隔离（非知识库级别）
index_name = f"ragflow_doc_meta_{tenant_id}"

# 区别于切片索引（按知识库隔离）
chunk_index = f"ragflow_chunk_{tenant_id}_{kb_id}"
```

### 存储格式
```json
{
  "id": "doc_id_xxx",
  "kb_id": "kb_id_yyy",
  "meta_fields": {
    "author": "张三",
    "tags": ["技术文档", "产品手册"],
    "version": "1.0",
    "date": "2024-01-15"
  }
}
```

**设计要点**:
- `id` 字段存储文档 ID（非自增 ID，而是 Document.id）
- `kb_id` 字段用于跨知识库查询时过滤
- `meta_fields` 为 JSON 对象，支持任意 schema
- 索引按租户级别创建，而非每个知识库一个索引（优化索引数量）

### 多后端支持

| 后端 | 索引类型 | 部分更新 | 特殊处理 |
|-----|---------|---------|---------|
| Elasticsearch | Index | ✅ `replace_meta_fields` 脚本 | 需手动 `refresh_idx` |
| Infinity | Table | ❌ delete+insert | 自动刷新，无需手动 |
| OceanBase | Table | ❌ delete+insert | 特殊 `SearchResult` 格式 |
| GaussDB | Table | ✅ 原生 insert...on conflict | - |

## 流程 1: 插入元数据

### 触发时机
- 文档上传后首次解析完成
- LLM 提取元数据完成
- Connector 批量导入文档

### 执行步骤

#### 步骤 1: 获取文档和租户信息
```python
doc_query = Document.select(Document, Knowledgebase.tenant_id)\
    .join(Knowledgebase, on=(Knowledgebase.id == Document.kb_id))\
    .where(Document.id == doc_id)
doc = doc_query.first()
tenant_id = doc.knowledgebase.tenant_id
```

**关键点**: 
- 使用 JOIN 获取租户 ID（避免二次查询）
- 租户 ID 决定元数据存储的索引名称

#### 步骤 2: 元数据预处理（分割合并值）
```python
def _split_combined_values(meta_fields: dict) -> dict:
    # 示例: "关羽、孙权、张辽" → ["关羽", "孙权", "张辽"]
    for key, value in meta_fields.items():
        if isinstance(value, list):
            new_values = []
            for item in value:
                if isinstance(item, str):
                    # 按中文顿号、逗号、分号、竖线分割
                    split_items = re.split(r"[、,，;；|]+", item.strip())
                    new_values.extend([s.strip() for s in split_items if s.strip()])
            processed[key] = dedupe_list(new_values)  # 去重
```

**设计意图**: 修复 LLM 提取时把多个值合并为一个字符串的问题

#### 步骤 3: 确保索引存在
```python
index_name = f"ragflow_doc_meta_{tenant_id}"
if not settings.docStoreConn.index_exist(index_name, kb_id):
    result = settings.docStoreConn.create_doc_meta_idx(index_name)
    if result is False:
        return False
```

**索引创建时机**: 
- 第一次插入元数据时自动创建
- 按租户级别创建（非知识库级别）

#### 步骤 4: 插入到向量存储
```python
doc_meta = {
    "id": doc_id,
    "kb_id": kb_id,
    "meta_fields": processed_meta_fields
}
result = settings.docStoreConn.insert(
    [doc_meta], 
    index_name, 
    kb_id,
    refresh="wait_for" if refresh_now else False
)
```

**参数说明**:
- `refresh_now=True`: 阻塞等待索引刷新，元数据立即可查（常规调用）
- `refresh_now=False`: 异步刷新，避免阻塞 Connector 批量导入

#### 步骤 5: 手动刷新索引（ES/OpenSearch）
```python
if refresh_now and not settings.DOC_ENGINE_INFINITY:
    settings.docStoreConn.refresh_idx(index_name)
```

**刷新策略**:
- Elasticsearch/OpenSearch: 需显式刷新才能立即搜索
- Infinity: 自动刷新,跳过此步骤

## 流程 2: 更新元数据

### 触发时机
- 用户手动编辑文档元数据
- LLM 重新提取元数据
- 批量更新操作 (`PATCH /datasets/{id}/documents/metadatas`)

### 执行策略（按后端差异）

#### Elasticsearch 策略: 脚本化部分更新
```python
# 检查文档是否存在
doc_exists = settings.docStoreConn.get(doc_id, index_name, [])
if doc_exists:
    # 调用后端的 replace_meta_fields 方法（完全替换）
    settings.docStoreConn.replace_meta_fields(index_name, doc_id, new_meta)
```

**关键点**:
- 使用脚本化更新 `ctx._source.meta_fields = params.new_meta`
- 完全替换而非深度合并（避免残留旧键）
- 如果后端不支持脚本，回退到 delete+insert

#### Infinity/OceanBase/SereneDB 策略: Delete + Insert
```python
cls.delete_document_metadata(doc_id, kb_id, tenant_id)
cls.insert_document_metadata(doc_id, new_meta, refresh_now=refresh_now)
```

**原因**: 这些后端不支持部分更新，必须全量替换

#### GaussDB 策略: Insert...On Conflict
```python
settings.docStoreConn.insert(
    [{"id": doc_id, "kb_id": kb_id, "meta_fields": new_meta}],
    index_name,
    kb_id
)
```

**优势**: 原生 UPSERT 语义，性能最优

### 特殊场景

#### 场景 1: 元数据变为空
```python
if not new_meta:
    cls.delete_document_metadata(doc_id, kb_id, tenant_id)
```

**设计**: 空元数据时删除整行，避免保留无意义记录

#### 场景 2: 文档不存在元数据行
```python
doc_exists = settings.docStoreConn.get(doc_id, index_name, [])
if not doc_exists:
    return cls.insert_document_metadata(doc_id, new_meta)
```

**回退逻辑**: 更新操作自动转为插入操作

## 流程 3: 删除元数据

### 触发时机
- 文档被删除
- 文档重新解析前清理旧数据
- 批量删除操作

### 执行步骤

#### 步骤 1: 检查索引是否存在（性能优化）
```python
if not settings.docStoreConn.index_exist(index_name, ""):
    return True  # 索引不存在 = 无元数据 = 删除成功
```

**优化点**: 避免对不存在的索引执行无效删除操作

#### 步骤 2: 确认元数据行存在
```python
existing_metadata = settings.docStoreConn.get(doc_id, index_name, [])
if not existing_metadata:
    return True  # 元数据不存在，视为成功
```

**优化点**: 避免删除不存在的记录（减少后端负载）

#### 步骤 3: 执行删除
```python
deleted_count = settings.docStoreConn.delete(
    {"id": doc_id},
    index_name,
    kb_id
)
```

**注意**: 元数据表是按租户级别的，`kb_id` 参数会被后端特殊处理（实际不作为过滤条件）

#### 步骤 4: 检查并删除空表（自动清理）
```python
def _drop_empty_metadata_table(index_name: str, tenant_id: str):
    # 优先使用原生 count API (ES/OpenSearch)
    count_value = settings.docStoreConn.count_idx(index_name)
    if count_value == 0:
        settings.docStoreConn.delete_idx(index_name, "")
```

**清理策略**:
- 删除文档后检查表是否为空
- 使用 `count_idx` API（快速）而非全表扫描
- 回退方案: `search(limit=1)` 检测是否有任何记录
- 防止空表堆积（优化租户资源占用）

## 流程 4: 批量更新元数据

### 入口
`PATCH /datasets/{id}/documents/metadatas`

### 请求格式
```json
{
  "selector": {
    "document_ids": ["doc1", "doc2"],
    "metadata_condition": {
      "logic": "and",
      "conditions": [
        {"key": "author", "operator": "eq", "value": "张三"}
      ]
    }
  },
  "updates": [
    {"key": "status", "value": "已审核"},
    {"key": "tags", "value": "新标签", "match": "旧标签"}
  ],
  "deletes": [
    {"key": "obsolete_field"},
    {"key": "tags", "value": "废弃标签"}
  ]
}
```

### 执行步骤

#### 步骤 1: 解析选择器（确定目标文档集）
```python
target_doc_ids = set()

# 分支 1: 按文档 ID 列表
if document_ids:
    kb_doc_ids = KnowledgebaseService.list_documents_by_ids([dataset_id])
    invalid_ids = set(document_ids) - set(kb_doc_ids)
    if invalid_ids:
        return error("这些文档不属于该知识库")
    target_doc_ids = set(document_ids)

# 分支 2: 按元数据条件过滤
if metadata_condition:
    metas = DocMetadataService.get_flatted_meta_by_kbs([dataset_id])
    filtered_ids = meta_filter(metas, convert_conditions(metadata_condition))
    target_doc_ids = target_doc_ids & filtered_ids  # 交集
```

**已知 Bug**:
```python
# 仅提供 metadata_condition 时，初始 target_doc_ids 为空集
# 导致交集运算结果永远为空
if metadata_condition and not document_ids:
    target_doc_ids = filtered_ids  # 应该直接赋值而非交集
```

#### 步骤 2: 执行批量更新逻辑
```python
updated_docs = DocMetadataService.batch_update_metadata(
    dataset_id, 
    target_doc_ids, 
    updates, 
    deletes
)
```

**内部逻辑**:
1. 搜索目标文档的元数据行
2. 对每个文档应用 updates 和 deletes 操作
3. 如果文档没有元数据行且有 updates，自动创建新行

#### 步骤 3: Updates 操作语义

| 场景 | 操作 | 示例 |
|-----|------|------|
| 字段不存在 + 无 match | 创建字段 | `{"key": "author", "value": "李四"}` |
| 字段为列表 + 无 match | 追加到列表 | `tags: ["A"]` + `{"key": "tags", "value": "B"}` → `["A", "B"]` |
| 字段为列表 + 有 match | 替换匹配项 | `tags: ["A", "B"]` + `{"key": "tags", "value": "C", "match": "A"}` → `["C", "B"]` |
| 字段为标量 + 无 match | 覆盖值 | `author: "张三"` + `{"key": "author", "value": "李四"}` → `"李四"` |
| 字段为标量 + 有 match | 条件覆盖 | `status: "草稿"` + `{"key": "status", "value": "发布", "match": "草稿"}` → `"发布"` |

#### 步骤 4: Deletes 操作语义

| 场景 | 操作 | 示例 |
|-----|------|------|
| 字段为列表 + 无 value | 删除整个字段 | `{"key": "tags"}` 删除 `tags` 字段 |
| 字段为列表 + 有 value | 删除列表中的值 | `{"key": "tags", "value": "A"}` 从列表中移除 "A" |
| 字段为标量 + 无 value | 删除整个字段 | `{"key": "author"}` 删除 `author` 字段 |
| 字段为标量 + 有 value | 条件删除 | `{"key": "status", "value": "草稿"}` 仅当值为 "草稿" 时删除 |

#### 步骤 5: 去重与规范化
```python
# 列表值自动去重
processed[key] = dedupe_list(new_values)

# 空元数据删除整行
if not meta:
    cls.delete_document_metadata(doc_id, kb_id, tenant_id)
```

## 流程 5: 元数据查询与过滤

### 查询方式 1: 获取单个文档元数据
```python
def get_document_metadata(doc_id: str) -> dict:
    # 步骤 1: 从 MySQL 获取租户 ID
    doc = Document.select(Document, Knowledgebase.tenant_id)\
        .join(Knowledgebase)\
        .where(Document.id == doc_id).first()
    
    # 步骤 2: 从元数据索引获取
    index_name = f"ragflow_doc_meta_{tenant_id}"
    metadata_doc = settings.docStoreConn.get(doc_id, index_name, [])
    
    # 步骤 3: 提取 meta_fields 字段
    return _extract_metadata(metadata_doc)
```

### 查询方式 2: 获取知识库所有元数据（扁平化）
```python
def get_flatted_meta_by_kbs(kb_ids: list[str]) -> dict:
    # 返回格式: {field_name: {value: [doc_ids]}}
    # 示例: {"author": {"张三": ["doc1", "doc2"], "李四": ["doc3"]}}
```

**用途**:
- 元数据过滤（`metadata_condition`）
- 前端元数据选择器（获取所有可选值）
- 聚合统计

**分页策略**:
```python
page_size = 1000
offset = 0
while True:
    batch = settings.docStoreConn.search(..., offset=offset, limit=page_size)
    if len(batch) < page_size:
        break
    offset += page_size
```

**性能警告**: 超过 100,000 文档时会记录警告日志

### 查询方式 3: Push-down 元数据过滤

#### 触发条件
```python
# 用户提供 metadata_condition 时尝试 push-down
doc_ids = DocMetadataService.filter_doc_ids_by_meta_pushdown(
    kb_ids, 
    filters, 
    logic="and", 
    limit=10000
)
if doc_ids is None:
    # Push-down 失败，回退到内存过滤
    metas = DocMetadataService.get_flatted_meta_by_kbs(kb_ids)
    doc_ids = meta_filter(metas, conditions, logic)
```

#### Elasticsearch Push-down
```python
# 构建 ES DSL 查询
query_body = {
    "query": {
        "bool": {
            "must": [
                {"term": {"kb_id": kb_id}},
                {"term": {"meta_fields.author.keyword": "张三"}}
            ]
        }
    }
}
response = es_client.search(index=index_name, body=query_body)
```

**限制检测**:
- 查询结果超过 `limit=10000` 时返回 `None`（避免截断数据）
- 使用 `track_total_hits: True` 获取精确总数

#### Infinity Push-down
```python
# 构建 SQL WHERE 子句
sql_where = "kb_id IN ('kb1', 'kb2') AND JSON_EXTRACT(meta_fields, '$.author') = '张三'"
```

#### GaussDB Push-down
```python
# 使用 JSONB 操作符
sql_where = "kb_id = ANY($1) AND meta_fields->>'author' = '张三'"
```

**不支持 Push-down 的场景**:
- 复杂的嵌套条件（多层 OR/AND）
- 正则表达式操作符（某些后端）
- 数组包含操作（`contains_any`）

## 特殊场景与设计考量

### 场景 1: 租户级索引 vs 知识库级索引

**设计决策**: 元数据索引按租户隔离，而非知识库隔离

**原因**:
- 减少索引数量（单租户可能有数百个知识库）
- 跨知识库元数据查询更高效
- 降低 ES 集群管理开销

**代价**:
- 查询时必须显式过滤 `kb_id`
- 删除知识库时需清理该租户索引中的残留数据

### 场景 2: MySQL meta_fields 字段已废弃

**历史设计**: Document 表有 `meta_fields TEXT` 字段

**当前状态**:
- 字段保留但不再使用
- 所有元数据读写操作直接访问 ES/Infinity
- MySQL 仅存储核心字段（id, name, size, status 等）

**迁移风险**: 旧数据可能仅存在于 MySQL，需要迁移脚本

### 场景 3: 分割合并值的必要性

**问题来源**: LLM 提取元数据时常把多个值合并为一个字符串

**示例**:
```json
// LLM 输出（错误）
{"authors": "关羽、孙权、张辽"}

// 分割后（正确）
{"authors": ["关羽", "孙权", "张辽"]}
```

**处理策略**:
- 自动检测常见分隔符：中文顿号、逗号、分号、竖线
- 仅对字符串类型值执行分割
- 分割后自动去重

### 场景 4: 空表自动清理

**触发时机**: 删除文档元数据后

**清理流程**:
1. 检查表是否为空（优先使用 `count_idx` API）
2. 如果为空，删除整个索引/表
3. 防止租户积累大量空表

**异常处理**: 清理失败不影响删除操作的成功（仅记录日志）

### 场景 5: 并发更新竞态

**问题**: 多个请求同时更新同一文档的元数据

**当前行为**: 最后写入生效（Last-Write-Wins）

**风险**: 
- 更新 A: `tags: ["A"]` → `["A", "B"]`
- 更新 B: `tags: ["A"]` → `["A", "C"]`
- 最终结果可能是 `["A", "B"]` 或 `["A", "C"]`，取决于执行顺序

**缓解方案**: 
- 批量更新 API 使用事务（但跨文档无法保证原子性）
- 前端可以实现乐观锁（基于 version 字段）

## 性能特征

| 操作 | 数据库查询 | 索引操作 | 典型耗时 |
|-----|----------|---------|---------|
| 插入元数据 | 1 次 JOIN | 1 次 insert + refresh | 50-150ms |
| 更新元数据 (ES) | 1 次 JOIN | 1 次 get + 1 次 script update | 30-80ms |
| 更新元数据 (Infinity) | 1 次 JOIN | 1 次 delete + 1 次 insert | 60-120ms |
| 删除元数据 | 0-1 次 | 1 次 exist + 1 次 get + 1 次 delete | 20-60ms |
| 单文档查询 | 1 次 JOIN | 1 次 get | 10-30ms |
| 扁平化查询 (1000 文档) | 1 次 | N 次分页 search | 200-800ms |
| Push-down 过滤 | 1 次 | 1 次 search | 50-200ms |

**性能瓶颈**:
- `refresh_idx` 阻塞等待（ES 默认 1 秒刷新间隔）
- 大规模扁平化查询（>10 万文档）
- 未使用 Push-down 时的全量内存过滤

**优化建议**:
- Connector 批量导入使用 `refresh_now=False`
- 优先使用 Push-down 过滤而非内存过滤
- 元数据字段控制在合理数量（<50 个 key）
- 避免超长列表值（>1000 项）

## 相关流程

- [文档上传流程](upload-flow.md): 上传时可携带初始元数据
- [文档删除流程](delete-flow.md): 删除文档时同步清理元数据
- [解析控制流程](parse-control-flow.md): 重新解析会保留元数据

## 代码追溯

**主要文件**:
- `api/db/services/doc_metadata_service.py`: 元数据服务核心实现（1417 行）
- `api/apps/restful_apis/document_api.py`: `update_metadata` 端点 (1400-1458)
- `common/metadata_utils.py`: `dedupe_list` 去重工具
- `common/metadata_es_filter.py`: ES Push-down 查询构建
- `common/metadata_gaussdb_filter.py`: GaussDB Push-down 查询构建
- `common/doc_store/doc_store_base.py`: `OrderByExpr` 排序表达式

**关键调用链**:
```
PATCH /datasets/{id}/documents/metadatas
  └─> update_metadata(tenant_id, dataset_id)
      ├─> KnowledgebaseService.accessible (权限校验)
      ├─> [selector.document_ids] 验证文档归属
      ├─> [selector.metadata_condition]
      │   ├─> get_flatted_meta_by_kbs (扁平化元数据)
      │   └─> meta_filter (内存过滤)
      └─> batch_update_metadata
          ├─> _search_metadata (查询目标文档元数据)
          ├─> _apply_updates / _apply_deletes (应用变更)
          ├─> update_document_metadata (更新非空元数据)
          └─> delete_document_metadata (删除空元数据)
              └─> _drop_empty_metadata_table (清理空表)
```
