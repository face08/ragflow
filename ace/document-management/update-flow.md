# 文档更新流程

## 概述

RAGFlow 文档更新模块提供了四种核心更新操作：
1. **文档重命名**：修改文档显示名称，同步到关联文件和向量索引
2. **状态切换**：启用/禁用文档，控制检索可见性
3. **切片方法变更**：切换解析器（parser_id），触发重新解析
4. **重新解析**：清空已有切片和索引，重新执行解析流程

所有更新操作都通过 `PUT /datasets/{dataset_id}/documents/{doc_id}` 端点统一入口，根据请求体中的字段组合路由到不同的内部处理逻辑。

## 核心数据流

```
请求体字段组合
    ↓
字段识别与路由
    ↓
┌────────────┬──────────────┬──────────────┬────────────────┐
│  仅 name   │  仅 status   │ chunk_method │  其它 config   │
│    变更    │    变更      │    变更      │     变更       │
└─────┬──────┴──────┬───────┴──────┬───────┴────────┬───────┘
      ↓             ↓              ↓                ↓
  rename_only   status_only   chunk_method   parser_config
    流程          流程           流程           流程
      ↓             ↓              ↓                ↓
  同步 File    更新 ES        重置 + 重解析    更新 config
  更新 ES      available_int
```

## 阶段详解

### 阶段 1:请求验证与权限检查

**代码位置**: `api/apps/restful_apis/document_api.py:1129-1216`

1. **知识库访问权限验证**
   - 检查 `KnowledgebaseService.accessible(kb_id=dataset_id, user_id=tenant_id)`
   - 不可访问 → 返回错误 "You don't own the dataset {dataset_id}."

2. **文档存在性检查**
   - `DocumentService.get_by_id(doc_id)` 查询文档
   - 不存在 → 返回错误 "Document not found!"

3. **文档-知识库关联校验**
   - 验证 `doc.kb_id == dataset_id`
   - 不匹配 → 返回错误 "Document {doc_id} not in dataset {dataset_id}."

**关键设计**: 采用双重权限检查，先验证知识库归属，再验证文档归属，防止跨知识库操作

### 阶段 2:更新路由与字段分发

**代码位置**: `api/apps/restful_apis/document_api.py:1129-1216`

根据请求体中的字段组合，路由到四个独立的处理分支：

| 条件判断 | 路由目标 | 处理逻辑 |
|---------|---------|---------|
| `len(req) == 1 and "name" in req` | `update_document_name_only` | 仅重命名 |
| `len(req) == 1 and "status" in req` | `update_document_status_only` | 仅状态切换 |
| `"chunk_method" in req` | `update_chunk_method` | 切片方法变更 |
| 其他 | `update_parser_config` | 通用配置更新 |

**优先级规则**: `chunk_method` 变更优先级最高,即使请求体包含其他字段也优先处理切片方法变更

### 阶段 3A:文档重命名流程 (update_document_name_only)

**代码位置**: `api/apps/services/document_api_service.py:19-38`

**触发条件**: 请求体仅包含 `name` 字段

**执行步骤**:

1. **数据库更新**
   ```python
   DocumentService.update_by_id(document_id, {"name": req_doc_name})
   ```

2. **关联文件同步**
   - 通过 `File2DocumentService.get_by_document_id(document_id)` 查找关联 File 记录
   - 若存在,同步更新 `FileService.update_by_id(file.id, {"name": req_doc_name})`
   - **设计意图**: 保持 File 和 Document 名称一致性,下载时显示正确文件名

3. **向量索引更新**
   - 获取 `tenant_id = DocumentService.get_tenant_id(document_id)`
   - 重新分词:
     ```python
     title_tks = rag_tokenizer.tokenize(req_doc_name)
     title_sm_tks = rag_tokenizer.fine_grained_tokenize(title_tks)
     ```
   - 更新 ES/Infinity 文档字段:
     ```python
     es_body = {"docnm_kwd": req_doc_name, 
                "title_tks": title_tks,
                "title_sm_tks": title_sm_tks}
     settings.docStoreConn.update({"doc_id": document_id}, es_body, 
                                   search.index_name(tenant_id), doc.kb_id)
     ```
   - **索引字段说明**:
     - `docnm_kwd`: 精确匹配关键字
     - `title_tks`: 标准分词,用于全文检索
     - `title_sm_tks`: 细粒度分词,支持前缀匹配

**性能特征**: 轻量级操作,无需重解析,毫秒级完成

### 阶段 3B:状态切换流程 (update_document_status_only)

**代码位置**: `api/apps/services/document_api_service.py:269-283`

**触发条件**: 请求体仅包含 `status` 字段

**执行步骤**:

1. **状态变更检查**
   ```python
   if doc.status is None or (int(doc.status) != status):
   ```
   - 仅当状态实际改变时才执行更新

2. **数据库更新**
   ```python
   DocumentService.update_by_id(doc.id, {"status": str(status)})
   ```

3. **向量索引同步**
   ```python
   settings.docStoreConn.update(
       {"doc_id": doc.id, "must_not": {"exists": "compile_kwd"}},
       {"available_int": status},
       search.index_name(kb.tenant_id), 
       doc.kb_id
   )
   ```
   - **查询条件**: `must_not: {"exists": "compile_kwd"}` 排除已编译的 Graph RAG 节点
   - **更新字段**: `available_int` 控制检索可见性 (0=禁用, 1=启用)

**批量操作支持**: 另有 `POST /datasets/{dataset_id}/documents/batch-update-status` 端点支持批量状态切换

### 阶段 3C:切片方法变更流程 (update_chunk_method)

**代码位置**: `api/apps/services/document_api_service.py:40-76`

**触发条件**: 请求体包含 `chunk_method` 字段

**执行步骤**:

1. **解析器变更判断**
   ```python
   if req["chunk_method"] != doc.parser_id:
       # 触发重置流程
       reset_document_for_reparse(doc, tenant_id, 
                                  parser_id=req["chunk_method"], 
                                  pipeline_id="")
   ```

2. **Pipeline 清理判断**
   - 即使 parser_id 未变,若 `doc.pipeline_id` 非空,也需重置
   - 确保切换到标准解析流程时清空高级特性配置

3. **解析器配置合并**
   - 调用 `get_parser_config(req["chunk_method"], tenant_id)` 获取默认配置
   - 合并请求体中的其他配置项 (如 chunk_token_num, layout_recognize 等)
   - 更新到数据库: `DocumentService.update_parser_config(doc.id, config)`

**关键行为**: 切片方法变更 **强制触发重新解析**,已有切片和索引将被清空

### 阶段 4:重置与重新解析 (reset_document_for_reparse)

**代码位置**: `api/apps/services/document_api_service.py:78-102`

**触发场景**:
- 切片方法变更
- Pipeline 配置清理
- 显式的重新解析请求

**执行步骤**:

1. **状态重置**
   ```python
   update_fields = {
       "progress": 0,
       "progress_msg": "",
       "run": TaskStatus.UNSTART.value
   }
   if parser_id is not None:
       update_fields["parser_id"] = parser_id
   if pipeline_id is not None:
       update_fields["pipeline_id"] = pipeline_id
   DocumentService.update_by_id(doc.id, update_fields)
   ```

2. **计数器回滚** (关键事务操作)
   ```python
   release_reparse_counters(doc.id)
   ```
   - 代码位置: `api/db/services/document_counter_service.py:20-55`
   - **事务保护**: `with DB.atomic()` + `for_update()` 行锁
   - **回滚逻辑**:
     ```python
     # 读取文档当前计数
     fresh = Document.select().where(Document.id == doc_id).for_update().first()
     # 从文档中减去
     Document.update(
         token_num=Document.token_num - fresh.token_num,
         chunk_num=Document.chunk_num - fresh.chunk_num,
         process_duration=Document.process_duration - fresh.process_duration
     ).where((Document.id == fresh.id) & (Document.kb_id == fresh.kb_id)).execute()
     # 从知识库中减去
     kb = Knowledgebase.select().where(Knowledgebase.id == fresh.kb_id).for_update().first()
     Knowledgebase.update(
         token_num=Knowledgebase.token_num - fresh.token_num,
         chunk_num=Knowledgebase.chunk_num - fresh.chunk_num
     ).where(Knowledgebase.id == fresh.kb_id).execute()
     ```
   - **已知竞态**: Worker 在解析完成后写入最终计数的事务可能晚于本事务提交,导致计数偏差 (代码注释已说明)

3. **向量索引清理**
   ```python
   settings.docStoreConn.delete(
       {"doc_id": doc.id}, 
       search.index_name(tenant_id), 
       doc.kb_id
   )
   ```

4. **切片图片清理**
   ```python
   DocumentService.delete_chunk_images(doc, tenant_id)
   ```
   - 删除存储桶 `{kb_id}` 下的所有切片图片
   - 异常捕获: 图片删除失败仅记录日志,不影响主流程

### 阶段 3D:通用配置更新流程 (update_parser_config)

**代码位置**: `api/apps/restful_apis/document_api.py:1129-1216`

**触发条件**: 请求体不满足前三个分支条件

**支持的配置字段**:
- `chunk_token_num`: 切片 token 数量
- `layout_recognize`: 版面识别开关
- `raptor`: Raptor 配置
- `task_page_size`: 任务批次大小
- `llm_id`: LLM 模型 ID
- 其他 parser_config 内的所有字段

**执行逻辑**:
```python
DocumentService.update_parser_config(doc.id, req)
```

**重要说明**: 仅更新配置,**不触发重新解析**,需配合解析触发接口手动启动

## 批量状态更新流程

**端点**: `POST /datasets/{dataset_id}/documents/batch-update-status`

**代码位置**: `api/apps/restful_apis/document_api.py:1974-2092`

**请求体**:
```json
{
  "document_ids": ["id1", "id2"],
  "status": 1
}
```

**执行流程**:

1. **前置校验**
   - 知识库访问权限检查
   - `document_ids` 非空校验
   - 重复 ID 检查: `check_duplicate_ids(document_ids, "document")`

2. **文档归属验证**
   ```python
   kb_docs = KnowledgebaseService.get_by_id(dataset_id)
   kb_doc_ids = set([d.id for d in kb_docs])
   invalid_ids = set(document_ids) - kb_doc_ids
   ```
   - 若存在无效 ID,返回错误列表

3. **零切片文档过滤**
   ```python
   zero_chunk_docs = DocumentService.query(
       id__in=document_ids, 
       chunk_num=0
   )
   zero_chunk_ids = [d.id for d in zero_chunk_docs]
   valid_ids = [id for id in document_ids if id not in zero_chunk_ids]
   ```
   - **设计理念**: `chunk_num == 0` 的文档未完成解析,不允许状态切换
   - 错误码映射: `"3022"` → "Document store table missing."

4. **批量更新**
   - 数据库: `DocumentService.update_by_id` 循环调用
   - 向量索引: 批量 `docStoreConn.update`
   - 返回: `{"updated": n, "errors": [...]}`

**部分成功处理**: 即使部分文档更新失败,已成功的更新不会回滚

## 特殊场景与注意事项

### 场景 1: 并发更新冲突

**问题**: 两个请求同时更新同一文档的不同字段
- 请求 A: 更新 name
- 请求 B: 更新 status

**当前行为**: 最后写入胜出 (Last Write Wins),无乐观锁保护

**建议**: 前端应避免并发编辑同一文档

### 场景 2: 切片方法变更的数据丢失风险

**风险点**: `reset_document_for_reparse` 会清空:
- 所有已生成的切片
- 向量索引中的文档记录
- 切片关联的图片资源

**用户体验**: 前端应明确提示 "切换解析方式将清空现有切片,需重新解析"

### 场景 3: 计数器竞态条件

**场景**: Worker 正在写入最终计数 → 用户触发重新解析 → `release_reparse_counters` 执行

**结果**: 知识库的 `chunk_num`/`token_num` 可能出现负数或偏差

**缓解措施**:
- 代码注释已标注此竞态
- 建议在文档 `run` 状态为 `RUNNING` 时禁止切片方法变更

### 场景 4: Graph RAG 节点的状态更新

**特殊处理**: 状态更新的向量索引查询条件包含 `must_not: {"exists": "compile_kwd"}`

**原因**: Graph RAG 编译生成的节点文档不应受原始文档状态控制

**影响**: 禁用原始文档不会影响已编译的知识图谱节点

## 性能特征

| 操作类型 | 数据库写入 | 向量索引更新 | 存储操作 | 典型耗时 |
|---------|----------|------------|---------|---------|
| 重命名 | 1-2 次 | 1 次 | 无 | <50ms |
| 状态切换 | 1 次 | 1 次 | 无 | <50ms |
| 配置更新 | 1 次 | 无 | 无 | <20ms |
| 切片方法变更 | 2 次 | 1 次删除 | 删除切片图片 | 100-500ms |
| 批量状态更新 | N 次 | 1 次批量 | 无 | 50ms × N |

**优化建议**:
- 批量操作优于循环单次调用
- 重命名/状态切换可并发执行 (不同文档)
- 切片方法变更应串行执行 (同一文档)

## 相关接口

### 元数据配置更新
`PUT /datasets/{dataset_id}/documents/{doc_id}/metadata/config`

更新文档的元数据 schema 配置,存储在 `parser_config.metadata` 字段

### 解析触发
- `POST /documents/ingest`: 统一解析控制接口 (run/rerun/cancel)
- `POST /datasets/{dataset_id}/documents/parse`: 批量解析触发

详见 [解析控制流程](parse-control-flow.md)

## 代码追溯

**主要文件**:
- `api/apps/restful_apis/document_api.py`: REST API 端点定义
- `api/apps/services/document_api_service.py`: 业务逻辑实现
- `api/db/services/document_service.py`: 数据访问层
- `api/db/services/document_counter_service.py`: 计数器事务管理
- `api/db/services/file_service.py`: 文件关联同步

**关键函数调用链**:
```
update_document (REST)
  └─> update_document_name_only
      ├─> DocumentService.update_by_id
      ├─> FileService.update_by_id
      └─> docStoreConn.update (title_tks)
  └─> update_document_status_only
      ├─> DocumentService.update_by_id
      └─> docStoreConn.update (available_int)
  └─> update_chunk_method
      ├─> reset_document_for_reparse
      │   ├─> release_reparse_counters (事务)
      │   ├─> docStoreConn.delete
      │   └─> delete_chunk_images
      └─> update_parser_config
  └─> DocumentService.update_parser_config
```
