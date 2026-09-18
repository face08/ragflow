# 解析控制流程

## 概述

RAGFlow 的解析控制模块负责管理文档解析任务的完整生命周期，包括：
- **触发解析**：启动新文档的解析或重新解析已有文档
- **停止解析**：取消正在进行的解析任务
- **任务状态管理**：跟踪解析进度和状态转换
- **资源清理**：清理中间产物和回滚计数器

系统提供三个核心接口：
- `POST /documents/ingest`：统一解析控制入口 (支持 run/rerun/cancel)
- `POST /datasets/{dataset_id}/documents/parse`：批量解析触发
- `POST /datasets/{dataset_id}/documents/stop`：批量停止解析

## 任务状态机

```
     上传完成
        ↓
    UNSTART (未开始)
        ↓
      触发解析
        ↓
    RUNNING (执行中)
        ↓
    ┌───┴───┬────────┐
    ↓       ↓        ↓
  DONE   CANCEL    FAIL
 (完成)  (已取消)  (失败)
    ↓       ↓        ↓
  重新解析 ← ← ← ← ← ←
```

**状态定义** (`common/constants.py:TaskStatus`):
- `UNSTART`: 文档已上传，等待解析
- `RUNNING`: 解析任务正在执行
- `DONE`: 解析成功完成
- `CANCEL`: 用户主动取消
- `FAIL`: 解析失败
- `SCHEDULE`: 定时任务调度状态 (不常用)

## 核心数据流

```
用户请求
    ↓
权限验证
    ↓
文档状态检查
    ↓
┌──────────┬──────────┬──────────┐
│  run     │  rerun   │  cancel  │
│  启动    │  重解析  │  取消    │
└────┬─────┴────┬─────┴────┬─────┘
     ↓          ↓          ↓
  创建任务   清理旧数据   取消任务
     ↓          ↓          ↓
  Worker    清空索引    Redis 通知
  执行       ↓          ↓
           创建任务   更新状态
             ↓
          Worker
           执行
```

## 阶段详解

### 阶段 1：统一解析控制 (POST /documents/ingest)

**代码位置**: `api/apps/restful_apis/document_api.py:1461-1538`

**请求体**:
```json
{
  "doc_ids": ["id1", "id2"],
  "run": "RUNNING",  // or "CANCEL"
  "delete": false,   // 重解析时是否清空旧数据
  "apply_kb": false, // 是否应用知识库默认配置
  "user_id": "optional_llm_user_id"
}
```

**字段说明**:
- `run`: 目标状态，`"RUNNING"` 启动解析，`"CANCEL"` 取消解析
- `delete`: 配合 `run="RUNNING"` 使用，清空已有切片后重新解析
- `apply_kb`: 将知识库的 `llm_id`、`enable_metadata`、`metadata` 配置同步到文档
- `user_id`: OpenAI API 的 `user` 字段，用于 LLM 调用追踪

**执行流程**:

#### 步骤 1: 文档访问权限校验
```python
for doc_id in req["doc_ids"]:
    if not DocumentService.accessible(doc_id, user_id):
        return RetCode.AUTHENTICATION_ERROR, "no authorization"
```

#### 步骤 2: 状态字段初始化
```python
info = {"run": str(req["run"]), "progress": 0}
rerun_with_delete = str(req["run"]) == TaskStatus.RUNNING.value and req.get("delete", False)
if rerun_with_delete:
    info["progress_msg"] = ""
    info["chunk_num"] = 0
    info["token_num"] = 0
```

**重解析场景**: 当 `run="RUNNING"` 且 `delete=true` 时，清空进度信息和计数器

#### 步骤 3: 租户校验
```python
doc_tenant_id = DocumentService.get_tenant_id(doc_id)
if not doc_tenant_id:
    return RetCode.DATA_ERROR, "Tenant not found!"
```

#### 步骤 4: 取消分支 (run="CANCEL")
```python
if str(req["run"]) == TaskStatus.CANCEL.value:
    tasks = list(TaskService.query(doc_id=doc_id))
    has_unfinished_task = any((task.progress or 0) < 1 for task in tasks)
    
    if str(doc.run) in [TaskStatus.RUNNING.value, TaskStatus.CANCEL.value] or has_unfinished_task:
        cancel_all_task_of(doc_id)
        cancel_doc_msg = f"\n{datetime.now().strftime('%H:%M:%S')} Task stopped by user."
        info["progress_msg"] = (doc.progress_msg or "") + cancel_doc_msg
    else:
        return RetCode.DATA_ERROR, "Cannot cancel a task that is not in RUNNING status"
```

**关键逻辑**:
- 检查文档状态 (`doc.run`) 或未完成任务 (`task.progress < 1`)
- 调用 `cancel_all_task_of(doc_id)`:
  - 查询所有关联任务
  - 通过 Redis pub/sub 发送取消通知: `REDIS_CONN.publish(f"cancel_task_{kb_id}", task_id)`
- 追加取消时间戳到 `progress_msg`，保留历史记录

#### 步骤 5: 重解析清理 (delete=true)
```python
if req.get("delete", False):
    # 删除所有任务记录
    TaskService.filter_delete([Task.doc_id == doc_id])
    
    # 清理 Graph RAG 导航索引
    from rag.advanced_rag.knowlege_compile.dataset_nav import remove_dataset_nav_doc_sync
    remove_dataset_nav_doc_sync(doc_tenant_id, doc.kb_id, doc.id)
    
    # 删除向量索引
    if settings.docStoreConn.index_exist(search.index_name(doc_tenant_id), doc.kb_id):
        settings.docStoreConn.delete({"doc_id": doc_id}, search.index_name(doc_tenant_id), doc.kb_id)
```

**清理范围**:
- MySQL `task` 表的所有关联记录
- Graph RAG 编译生成的导航节点
- Elasticsearch/Infinity 中的所有切片

**不清理**:
- 原始文档文件 (存储桶中的 blob)
- `document` 表的元数据记录
- 切片图片 (由 `reset_document_for_reparse` 清理)

#### 步骤 6: 应用知识库配置 (apply_kb=true)
```python
if str(req["run"]) == TaskStatus.RUNNING.value:
    if req.get("apply_kb"):
        e, kb = KnowledgebaseService.get_by_id(doc.kb_id)
        if not e:
            raise LookupError("Can't find this dataset!")
        doc.parser_config["llm_id"] = kb.parser_config.get("llm_id")
        doc.parser_config["enable_metadata"] = kb.parser_config.get("enable_metadata", False)
        doc.parser_config["metadata"] = kb.parser_config.get("metadata", {})
        DocumentService.update_parser_config(doc.id, doc.parser_config)
```

**使用场景**: 文档创建后修改了知识库的默认配置，需要同步到该文档

#### 步骤 7: 启动解析任务
```python
doc_dict = doc.to_dict()
DocumentService.run(
    doc_tenant_id, 
    doc_dict, 
    kb_table_num_map, 
    user_id=normalize_llm_user_id(req.get("user_id"))
)
```

**`DocumentService.run` 职责**:
- 创建 `Task` 记录
- 将任务提交到异步 Worker 队列
- Worker 执行解析、切片、向量化流程

### 阶段 2: 批量解析触发 (POST /datasets/{dataset_id}/documents/parse)

**代码位置**: `api/apps/restful_apis/document_api.py:1541-1658`

**设计目标**: 提供更严格的批量解析接口，自动处理重解析场景

**请求体**:
```json
{
  "document_ids": ["id1", "id2", "id3"],
  "user_id": "optional_llm_user_id"
}
```

**执行流程**:

#### 步骤 1: 前置验证
```python
if not KnowledgebaseService.accessible(kb_id=dataset_id, user_id=tenant_id):
    return get_error_data_result(message=f"You don't own the dataset {dataset_id}.")

req = await get_request_json()
if not req:
    return get_error_data_result(message="Request body is required.")

document_ids = req.get("document_ids", [])
if not document_ids or not isinstance(document_ids, list):
    return get_error_data_result(message="document_ids is required and must be a non-empty list.")
```

#### 步骤 2: ID 重复检查
```python
check_duplicate_ids(document_ids, "document")
```
- 若存在重复 ID，返回错误列表但不中断流程

#### 步骤 3: 文档归属验证
```python
not_found_ids = []
valid_doc_ids = []
for doc_id in document_ids:
    docs = DocumentService.query(kb_id=dataset_id, id=doc_id)
    if not docs:
        not_found_ids.append(doc_id)
    else:
        valid_doc_ids.append(doc_id)
```

**部分失败处理**: 即使部分 ID 不存在，仍会解析有效的文档，最终返回错误列表

#### 步骤 4: 逐文档解析 (内部 _run_sync 逻辑)
```python
for doc_id in valid_doc_ids:
    e, doc = DocumentService.get_by_id(doc_id)
    
    # 重解析清理
    if str(doc.run) == TaskStatus.DONE.value:
        DocumentService.clear_chunk_num_when_rerun(doc.id)
        info = {"progress_msg": "", "chunk_num": 0, "token_num": 0}
    
    # 更新状态
    info["run"] = str(TaskStatus.RUNNING.value)
    info["progress"] = 0
    DocumentService.update_by_id(doc_id, info)
    
    # 清理旧数据
    TaskService.filter_delete([Task.doc_id == doc_id])
    remove_dataset_nav_doc_sync(tenant_id, doc.kb_id, doc.id)
    if settings.docStoreConn.index_exist(search.index_name(tenant_id), doc.kb_id):
        settings.docStoreConn.delete({"doc_id": doc_id}, search.index_name(tenant_id), doc.kb_id)
    
    # 启动解析
    DocumentService.run(tenant_id, doc.to_dict(), kb_table_num_map, user_id=llm_user_id)
    success_count += 1
```

**与 `/documents/ingest` 的差异**:
| 特性 | /documents/ingest | /documents/parse |
|-----|------------------|------------------|
| 清理控制 | 通过 `delete` 参数 | 自动清理 (DONE 状态文档) |
| 知识库配置同步 | 通过 `apply_kb` 参数 | 不支持 |
| 取消功能 | 支持 `run="CANCEL"` | 不支持 |
| 验证严格度 | 宽松 | 严格 (预先校验归属) |
| 使用场景 | 通用控制接口 | 批量重解析专用 |

#### 返回值结构
```json
{
  "code": 0,
  "data": {
    "success_count": 2,
    "errors": [
      "Document id3 not found in dataset"
    ]
  }
}
```

**注意**: 即使 `not_found_ids` 非空，`code` 仍可能为非零 (部分失败场景)

### 阶段 3: 批量停止解析 (POST /datasets/{dataset_id}/documents/stop)

**代码位置**: `api/apps/restful_apis/document_api.py:1661-1783`

**设计目标**: 安全地终止正在运行的解析任务，回滚已写入的部分计数

**请求体**:
```json
{
  "document_ids": ["id1", "id2"]
}
```

**执行流程**:

#### 步骤 1: 严格前置验证
```python
check_duplicate_ids(document_ids, "document")

# 验证所有文档归属
kb_docs = DocumentService.query(kb_id=dataset_id)
kb_doc_ids = set([d.id for d in kb_docs])
invalid_ids = set(document_ids) - kb_doc_ids

if invalid_ids:
    return get_error_data_result(
        message=f"These documents do not belong to dataset {dataset_id}: {', '.join(invalid_ids)}"
    )
```

**差异**: 与 `/documents/parse` 不同，此接口采用 **全部有效或全部拒绝** 策略，不允许部分失败

#### 步骤 2: 停止条件检查
```python
for doc_id in document_ids:
    e, doc = DocumentService.get_by_id(doc_id)
    tasks = list(TaskService.query(doc_id=doc_id))
    has_unfinished_task = any((task.progress or 0) < 1 for task in tasks)
    
    if str(doc.run) not in [TaskStatus.RUNNING.value, TaskStatus.CANCEL.value] and not has_unfinished_task:
        errors.append(f"Document {doc_id}: Can't stop parsing document that has not started or already completed")
        continue
```

**可停止状态**:
- `doc.run` 为 `RUNNING` 或 `CANCEL`
- 或存在未完成的任务 (`task.progress < 1`)

#### 步骤 3: 取消任务
```python
cancel_all_task_of(doc_id)
```

**取消机制** (`rag/utils/__init__.py`):
```python
def cancel_all_task_of(doc_id):
    tasks = list(TaskService.query(doc_id=doc_id))
    if not tasks:
        return
    kb_id = tasks[0].from_id
    for task in tasks:
        REDIS_CONN.publish(f"cancel_task_{kb_id}", task.id)
```

**Worker 响应**:
- Worker 订阅 `cancel_task_{kb_id}` 频道
- 收到任务 ID 后，设置内部取消标志
- 在下一个检查点停止处理并退出

#### 步骤 4: 计数器回滚 (关键事务)
```python
try:
    release_reparse_counters(doc_id)
except LookupError:
    logging.exception("Failed to release counters for document %s during stop-parse", doc_id)
    errors.append(f"Document not found: {doc_id}")
    continue
```

**`release_reparse_counters` 详细逻辑** (`api/db/services/document_counter_service.py:20-55`):
```python
def release_reparse_counters(doc_id):
    with DB.atomic():
        # 行锁读取
        fresh = Document.select().where(Document.id == doc_id).for_update().first()
        if fresh is None:
            raise LookupError(doc_id)
        
        if not (fresh.token_num or fresh.chunk_num or fresh.process_duration):
            return  # 无需回滚
        
        # 从文档中减去计数
        Document.update(
            token_num=Document.token_num - fresh.token_num,
            chunk_num=Document.chunk_num - fresh.chunk_num,
            process_duration=Document.process_duration - fresh.process_duration
        ).where((Document.id == fresh.id) & (Document.kb_id == fresh.kb_id)).execute()
        
        # 从知识库中减去计数
        kb = Knowledgebase.select().where(Knowledgebase.id == fresh.kb_id).for_update().first()
        if kb is None:
            raise LookupError("Knowledgebase not found which is supposed to be there")
        
        if fresh.token_num or fresh.chunk_num:
            Knowledgebase.update(
                token_num=Knowledgebase.token_num - fresh.token_num,
                chunk_num=Knowledgebase.chunk_num - fresh.chunk_num
            ).where(Knowledgebase.id == fresh.kb_id).execute()
```

**事务保护**:
- `DB.atomic()` 确保原子性
- `for_update()` 行锁防止并发修改
- 同时回滚文档和知识库的计数

**已知竞态** (代码注释):
> This does not serialize a worker that writes its final counts in a separate 
> transaction after the release commits; fully closing the stop-parse-during-parse 
> race needs a worker-side cancel check.

**影响**: Worker 可能在计数回滚后才提交最终计数，导致知识库统计出现偏差

#### 步骤 5: 状态更新与标记
```python
cancel_doc_msg = f"\n{datetime.now().strftime('%H:%M:%S')} Task stopped by user."
DocumentService.update_by_id(doc_id, {
    "run": str(TaskStatus.CANCEL.value),
    "progress": 0,
    "progress_msg": (doc.progress_msg or "") + cancel_doc_msg
})
```

**设计细节**:
- `progress` 重置为 0
- `progress_msg` 追加而非覆盖，保留完整历史
- **不修改** `chunk_num`/`token_num`，已由 `release_reparse_counters` 回滚

#### 步骤 6: 向量索引清理
```python
index_name = search.index_name(tenant_id)
if settings.docStoreConn.index_exist(index_name, doc.kb_id):
    settings.docStoreConn.delete({"doc_id": doc.id}, index_name, doc.kb_id)
```

**清理范围**: 删除已写入的部分切片，避免不完整数据影响检索

## 特殊场景与边界条件

### 场景 1: 重复解析请求
**问题**: 文档状态已是 `RUNNING`，用户再次调用 `/documents/parse`

**当前行为**:
1. 不会检测重复
2. 清空旧任务记录: `TaskService.filter_delete([Task.doc_id == doc_id])`
3. 创建新任务，旧任务的 Worker 继续运行但无法更新进度 (任务记录已删除)

**建议**: 前端应在 `doc.run == "RUNNING"` 时禁用"解析"按钮

### 场景 2: 取消后立即重新解析
**操作序列**:
1. 用户停止解析: `POST /documents/stop`
2. 立即触发解析: `POST /documents/parse`

**风险点**:
- Redis 取消通知是异步的，Worker 可能尚未响应
- 新任务可能与旧 Worker 并发执行

**缓解措施**:
- `TaskService.filter_delete` 删除旧任务记录
- Worker 更新进度时会因任务不存在而失败退出

### 场景 3: 计数器竞态的修复方案
**问题根源**: `release_reparse_counters` 的事务与 Worker 写入最终计数的事务不串行

**可能后果**:
- 知识库 `chunk_num` 变为负数
- 或计数少于实际切片数

**完整修复** (需 Worker 侧改动):
```python
# Worker 侧在写入最终计数前
if should_cancel():  # 检查 Redis 取消信号
    return  # 不写入计数，由 release_reparse_counters 统一处理
```

**当前缓解**:
- 代码注释已标注此问题
- 建议在文档 `run == "RUNNING"` 时灰化"切换解析器"和"停止"按钮

### 场景 4: 部分文档停止失败
**场景**: 批量停止 10 个文档，其中 2 个已完成 (`run == "DONE"`)

**当前行为**:
```python
for doc_id in document_ids:
    # 检查失败 → 添加到 errors，continue
    if not can_stop(doc):
        errors.append(f"Document {doc_id}: Can't stop...")
        continue
    # 其他文档继续处理
```

**返回值**:
```json
{
  "code": 0,
  "data": {
    "success_count": 8,
    "errors": [
      "Document xxx: Can't stop parsing document that has not started or already completed",
      "Document yyy: Can't stop parsing document that has not started or already completed"
    ]
  }
}
```

**前端建议**: 展示成功/失败统计，而非全有全无的提示

## 性能与资源特征

| 操作 | 数据库事务 | Redis 操作 | 向量索引 | 典型耗时 |
|-----|----------|-----------|---------|---------|
| 触发解析 | 2 次写入 | 无 | 无 | 50-100ms |
| 重解析 (delete=true) | 3 次写入 | 无 | 1 次删除 | 100-300ms |
| 取消解析 | 2 次更新 | N 次 pub | 无 | 50ms |
| 停止解析 | 3 次 (含事务锁) | N 次 pub | 1 次删除 | 100-200ms |

**性能瓶颈**:
- `release_reparse_counters` 的 `for_update()` 行锁可能阻塞其他更新
- 向量索引删除操作在大量切片时较慢 (ES 刷新延迟)

**优化建议**:
- 批量操作优于循环单次调用
- 避免高频重复解析同一文档
- 停止解析后等待 2-3 秒再重新解析，让 Worker 完全退出

## 错误处理与恢复

### 常见错误码

| 错误信息 | 原因 | 恢复方案 |
|---------|------|---------|
| "You don't own the dataset" | 知识库权限不足 | 检查用户归属 |
| "Tenant not found!" | 租户 ID 丢失 | 数据一致性问题，联系管理员 |
| "Cannot cancel a task that is not in RUNNING status" | 文档未在运行 | 刷新状态后重试 |
| "Document not found!" | 文档已被删除 | 从列表中移除 |
| "Can't stop parsing document that has not started or already completed" | 状态不可停止 | 检查 `doc.run` 状态 |

### 状态不一致修复

**症状**: 文档显示 `RUNNING` 但实际无任务

**诊断**:
```python
doc = DocumentService.get_by_id(doc_id)
tasks = list(TaskService.query(doc_id=doc_id))
print(f"doc.run={doc.run}, tasks={len(tasks)}, has_unfinished={any(t.progress < 1 for t in tasks)}")
```

**手动修复**:
```python
# 方案 1: 强制重置状态
DocumentService.update_by_id(doc_id, {"run": TaskStatus.UNSTART.value, "progress": 0})

# 方案 2: 触发重新解析
# POST /documents/parse with delete=true
```

## 接口对比总结

| 特性 | POST /documents/ingest | POST /documents/parse | POST /documents/stop |
|-----|----------------------|---------------------|---------------------|
| **主要用途** | 通用控制入口 | 批量重解析 | 批量停止 |
| **取消功能** | ✅ `run="CANCEL"` | ❌ | ✅ 专用 |
| **清理控制** | 可选 `delete` | 自动清理 DONE 文档 | 自动清理 |
| **知识库配置同步** | 可选 `apply_kb` | ❌ | N/A |
| **失败策略** | 部分成功 | 部分成功 | 全部成功或全部失败 |
| **ID 验证时机** | 执行时 | 预先验证 | 预先验证 |
| **适用场景** | 单文档精细控制 | 批量自动化 | 紧急停止 |

## 相关流程

### 文档上传后的自动解析
上传接口 (`POST /datasets/{id}/documents`) 不会自动触发解析，需显式调用解析接口

### 切片方法变更触发的重解析
详见 [文档更新流程](update-flow.md) 的 `reset_document_for_reparse` 章节

### 解析进度监控
通过 `GET /datasets/{id}/documents` 查询 `run`、`progress`、`progress_msg` 字段

## 代码追溯

**主要文件**:
- `api/apps/restful_apis/document_api.py`: REST API 端点
- `api/db/services/document_service.py`: `run()` 方法启动解析
- `api/db/services/task_service.py`: 任务记录管理
- `api/db/services/document_counter_service.py`: `release_reparse_counters()` 事务
- `rag/utils/__init__.py`: `cancel_all_task_of()` 取消逻辑
- `rag/advanced_rag/knowlege_compile/dataset_nav.py`: `remove_dataset_nav_doc_sync()`

**关键调用链**:
```
POST /documents/ingest
  └─> _run_sync(user_id, req)
      ├─> [CANCEL 分支]
      │   ├─> cancel_all_task_of(doc_id)
      │   └─> DocumentService.update_by_id (状态更新)
      ├─> [delete=true 分支]
      │   ├─> TaskService.filter_delete
      │   ├─> remove_dataset_nav_doc_sync
      │   └─> docStoreConn.delete
      └─> [RUNNING 分支]
          ├─> [apply_kb=true] update_parser_config
          └─> DocumentService.run
              └─> Worker 异步执行

POST /datasets/{id}/documents/stop
  └─> for doc_id in document_ids:
      ├─> cancel_all_task_of(doc_id)
      ├─> release_reparse_counters(doc_id) [事务]
      ├─> DocumentService.update_by_id (CANCEL 状态)
      └─> docStoreConn.delete (部分切片)
```
