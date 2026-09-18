# 文档删除流程详解

## 概述

本文档详细分析 RAGFlow 项目中文档删除功能的实现逻辑和执行流程。删除操作涉及多个层次的清理工作：取消正在进行的任务、清理索引数据、删除数据库记录、解除文件关联以及清理对象存储。

## 核心文件

- **API 层**: `api/apps/restful_apis/document_api.py`
- **Service 层**: 
  - `api/db/services/document_service.py` - 文档删除核心逻辑
  - `api/db/services/file_service.py` - 文件存储清理
  - `api/db/services/file2document_service.py` - 关联关系管理
- **数据模型**: `api/db/db_models.py`

---

## 删除流程的 10 个阶段

### 阶段 1: API 请求接收与验证

**位置**: `api/apps/restful_apis/document_api.py:1129-1216` (`delete_documents()`)

**功能**:
- 接收 DELETE 请求: `/datasets/<dataset_id>/documents`
- 解析请求参数 (document_ids)
- 验证用户权限和 tenant_id
- 检查 dataset_id 的有效性

**关键代码**:
```python
@document_bp.route("/datasets/<dataset_id>/documents", methods=["DELETE"])
@token_required
async def delete_documents(dataset_id, tenant_id):
    req = await request.json
    document_ids = req.get("document_ids", [])
    
    if not document_ids:
        return get_error_data_result(
            message="Please select at least one document to delete.",
            code=RetCode.ARGUMENT_ERROR
        )
```

### 阶段 2: 文档记录查询与权限确认

**位置**: `api/apps/restful_apis/document_api.py:1151-1174`

**功能**:
- 根据 document_ids 和 kb_id 查询文档记录
- 确认文档属于指定的 dataset
- 验证文档所有权 (tenant_id 匹配)

**关键代码**:
```python
docs = DocumentService.get_by_kb_ids([dataset_id], document_ids, tenant_id)
if not docs:
    return get_error_data_result(
        message="Documents not found or access denied.",
        code=RetCode.DATA_ERROR
    )
```

### 阶段 3: 取消正在运行的解析任务

**位置**: `api/db/services/document_service.py:463-572` (`remove_document()`)

**功能**:
- 检查文档的 TaskStatus
- 如果状态为 RUNNING,取消正在进行的解析任务
- 通过 redis pub/sub 通知 worker 停止处理

**关键代码**:
```python
def remove_document(doc, tenant_id):
    # Cancel running tasks
    if doc.status == TaskStatus.RUNNING.value:
        # Publish cancel message to redis
        REDIS_CONN.publish(
            f"cancel_task_{doc.kb_id}",
            json.dumps({"doc_id": doc.id})
        )
```

**特殊场景**: 正在解析的文档
- 状态为 `TaskStatus.RUNNING` 的文档会先被取消
- Worker 进程会收到取消信号并停止当前任务
- 避免产生孤儿任务或资源泄漏

### 阶段 4: 清理 Elasticsearch 索引

**位置**: `api/db/services/document_service.py:520-535`

**功能**:
- 从 Elasticsearch/Infinity 删除文档的所有 chunk 数据
- 使用 kb_id 作为索引名称
- 按 doc_id 批量删除

**关键代码**:
```python
# Delete chunks from ES/Infinity
try:
    ELASTICSEARCH.deleteByQuery(
        Q("match", doc_id=doc.id),
        index_names=[doc.kb_id]
    )
except Exception as e:
    logging.warning(f"ES delete failed for doc {doc.id}: {e}")
    # Continue deletion even if ES cleanup fails
```

**容错机制**:
- ES 删除失败不会中断整个删除流程
- 记录警告日志供后续手动清理
- 保证数据库和存储的一致性优先

### 阶段 5: 删除数据库中的 Document 记录

**位置**: `api/db/services/document_service.py:540-548`

**功能**:
- 从 Document 表删除主记录
- 触发级联删除相关的元数据

**关键代码**:
```python
def remove_document(doc, tenant_id):
    # Delete document record
    num_deleted = DocumentService.delete_by_ids([doc.id], tenant_id)
    if num_deleted == 0:
        raise RuntimeError(f"Failed to delete document {doc.id}")
```

### 阶段 6: 解除 File2Document 关联

**位置**: `api/db/services/file2document_service.py:70-71` (`delete_by_document_id()`)

**功能**:
- 删除 File2Document 表中的关联记录
- 维护 File 和 Document 的多对多关系
- 为后续文件清理提供孤立文件列表

**关键代码**:
```python
@classmethod
@DB.connection_context()
def delete_by_document_id(cls, document_id):
    return cls.model.delete().where(
        cls.model.document_id == document_id
    ).execute()
```

### 阶段 7: 识别孤立的 File 记录

**位置**: `api/db/services/file_service.py:757-798` (`delete_docs()`)

**功能**:
- 查找不再被任何 Document 引用的 File 记录
- 使用 LEFT JOIN 找出孤立文件
- 统计需要清理的文件数量

**关键代码**:
```python
# Find orphaned files
orphaned_files = (
    File.select()
    .join(File2Document, JOIN.LEFT_OUTER, on=(File.id == File2Document.file_id))
    .where(
        (File.tenant_id == tenant_id) &
        (File2Document.id.is_null())
    )
)
```

**去重机制**:
- 通过 `content_hash` 字段进行内容去重
- 相同内容的文件在存储中只保存一份
- 删除时需要检查引用计数,避免误删共享文件

### 阶段 8: 从对象存储删除文件

**位置**: `api/db/services/file_service.py:780-790`

**功能**:
- 调用 MinIO/S3 API 删除物理文件
- 使用 kb_id 作为 bucket 名称
- 使用 File.location 作为对象 key

**关键代码**:
```python
def delete_docs(doc_ids, tenant_id):
    # Get storage addresses
    for f2d in File2DocumentService.get_storage_address(doc_ids):
        bucket = f2d["storage_bucket"]
        location = f2d["storage_location"]
        
        try:
            # Delete from object storage
            settings.STORAGE_IMPL.rm(bucket, location)
            deleted_file_count += 1
        except Exception as e:
            logging.error(f"Failed to delete {bucket}/{location}: {e}")
            # Continue with next file
```

**错误隔离**:
- 单个文件删除失败不影响其他文件
- 记录详细的错误日志
- 最终返回成功删除的文件数量

### 阶段 9: 清理 File 数据库记录

**位置**: `api/db/services/file_service.py:793-798`

**功能**:
- 删除 File 表中的孤立记录
- 清理缩略图、元数据等附属信息

**关键代码**:
```python
# Delete orphaned file records
if deleted_file_count > 0:
    FileService.delete_by_document_ids(doc_ids, tenant_id)
```

### 阶段 10: 返回删除结果

**位置**: `api/apps/restful_apis/document_api.py:1195-1216`

**功能**:
- 统计删除的文档数量
- 构建响应数据
- 返回成功状态码

**关键代码**:
```python
@document_bp.route("/datasets/<dataset_id>/documents", methods=["DELETE"])
async def delete_documents(dataset_id, tenant_id):
    # ... deletion logic ...
    
    return get_json_result(
        data={
            "deleted_count": len(document_ids),
            "deleted_ids": document_ids
        }
    )
```

---

## 特殊场景处理

### 1. 删除正在解析的文档

**场景**: 用户删除状态为 `TaskStatus.RUNNING` 的文档

**处理流程**:
1. 通过 redis pub/sub 发送取消信号
2. Worker 进程收到信号后停止解析
3. 等待任务状态变为 CANCEL 或超时
4. 继续执行删除流程

**代码位置**: `api/db/services/document_service.py:490-510`

### 2. 删除 TABLE 类型文档

**场景**: 文档类型为 TABLE (Excel/CSV 等)

**特殊处理**:
- TABLE 文档可能有额外的表格数据存储
- 需要同时清理原始文件和解析后的表格数据
- 涉及额外的数据库表清理

**代码位置**: `api/db/services/document_service.py:550-560`

### 3. 批量删除中的部分失败

**场景**: 删除多个文档时部分失败

**容错机制**:
```python
failed_ids = []
success_ids = []

for doc_id in document_ids:
    try:
        DocumentService.remove_document(doc_id, tenant_id)
        success_ids.append(doc_id)
    except Exception as e:
        logging.error(f"Failed to delete doc {doc_id}: {e}")
        failed_ids.append(doc_id)

# Return partial success
return {
    "success_ids": success_ids,
    "failed_ids": failed_ids,
    "total": len(document_ids)
}
```

### 4. 内容去重文件的删除

**场景**: 多个 Document 共享同一个 File (相同 content_hash)

**处理逻辑**:
- 检查 File2Document 关联数量
- 只有当最后一个引用被删除时才清理物理文件
- 保证其他文档不受影响

**代码位置**: `api/db/services/file_service.py:770-778`

---

## 事务与一致性保证

### 数据库事务

```python
@DB.connection_context()
def remove_document(doc, tenant_id):
    with DB.atomic():
        # All DB operations in one transaction
        DocumentService.delete_by_ids([doc.id], tenant_id)
        File2DocumentService.delete_by_document_id(doc.id)
        # ... other DB operations ...
```

### 最终一致性策略

由于涉及多个存储系统 (数据库、ES、对象存储),无法保证强一致性:

1. **优先保证数据库一致性**: 使用事务确保关系数据完整
2. **ES 清理失败可容忍**: 记录日志,后续手动清理或通过定时任务清理
3. **对象存储清理失败可容忍**: 孤立文件不会影响系统功能,可后续回收
4. **删除顺序设计**: 先删索引和存储,最后删数据库记录,避免数据孤岛

---

## 性能优化建议

### 1. 批量删除优化

**当前**: 循环调用单文档删除
**优化**: 批量 ES 删除、批量存储删除

```python
# Batch ES deletion
ELASTICSEARCH.deleteByQuery(
    Q("terms", doc_id=doc_ids),  # Use 'terms' instead of multiple 'match'
    index_names=[kb_id]
)

# Batch storage deletion
settings.STORAGE_IMPL.rm_batch(bucket, locations)  # If supported
```

### 2. 异步删除

**当前**: 同步等待所有删除操作完成
**优化**: 将耗时的存储删除放入后台队列

```python
# Immediate response
@document_bp.route("/datasets/<dataset_id>/documents", methods=["DELETE"])
async def delete_documents(dataset_id, tenant_id):
    # Quick DB deletion
    DocumentService.mark_as_deleted(document_ids)
    
    # Enqueue background cleanup
    deletion_queue.enqueue("cleanup_storage", document_ids)
    
    return success_response()
```

### 3. 索引删除性能

**优化点**:
- 使用 bulk API 批量删除 ES 文档
- 考虑使用 `delete_by_query` 的 `wait_for_completion=false` 参数
- 定期运行 ES force merge 回收删除的空间

---

## 错误处理

### 常见错误场景

| 错误类型 | 原因 | 处理方式 |
|---------|------|---------|
| DocumentNotFound | 文档不存在或已被删除 | 返回 404,记录日志 |
| PermissionDenied | tenant_id 不匹配 | 返回 403,拒绝删除 |
| ESDeleteFailed | Elasticsearch 连接失败 | 记录警告,继续删除 DB |
| StorageDeleteFailed | MinIO/S3 删除失败 | 记录错误,标记为待清理 |
| DBConstraintViolation | 外键约束冲突 | 回滚事务,返回 500 |

### 错误日志示例

```python
logging.error(
    f"Failed to delete document",
    extra={
        "doc_id": doc.id,
        "kb_id": doc.kb_id,
        "tenant_id": tenant_id,
        "error": str(e),
        "stack_trace": traceback.format_exc()
    }
)
```

---

## 完整代码示例

### 核心删除逻辑 (简化版)

```python
# api/db/services/document_service.py
@classmethod
@DB.connection_context()
def remove_document(cls, doc, tenant_id):
    """
    完整删除一个文档:
    1. 取消运行中的任务
    2. 清理 ES 索引
    3. 删除 DB 记录
    4. 清理存储文件
    """
    
    # Step 1: Cancel running tasks
    if doc.status == TaskStatus.RUNNING.value:
        REDIS_CONN.publish(
            f"cancel_task_{doc.kb_id}",
            json.dumps({"doc_id": doc.id})
        )
        # Wait for cancellation (with timeout)
        wait_for_task_cancel(doc.id, timeout=30)
    
    # Step 2: Delete from Elasticsearch
    try:
        ELASTICSEARCH.deleteByQuery(
            Q("match", doc_id=doc.id),
            index_names=[doc.kb_id]
        )
    except Exception as e:
        logging.warning(f"ES delete failed for doc {doc.id}: {e}")
    
    # Step 3: Delete database records (in transaction)
    with DB.atomic():
        # Delete File2Document associations
        File2DocumentService.delete_by_document_id(doc.id)
        
        # Delete Document record
        num_deleted = cls.delete_by_ids([doc.id], tenant_id)
        if num_deleted == 0:
            raise RuntimeError(f"Failed to delete document {doc.id}")
    
    # Step 4: Clean up orphaned files from storage
    FileService.delete_docs([doc.id], tenant_id)
    
    logging.info(f"Successfully deleted document {doc.id}")
```

---

## 与上传/更新流程的对比

| 方面 | 上传流程 | 删除流程 |
|-----|---------|---------|
| **复杂度** | 高 (涉及解析、chunk、索引) | 中 (主要是清理工作) |
| **事务性** | 弱一致性 (异步处理) | 强一致性 (同步删除) |
| **可回滚** | 部分可回滚 | 不可回滚 (永久删除) |
| **性能瓶颈** | 解析和 OCR | 存储删除和 ES 查询 |
| **错误处理** | 重试机制 | 容错继续,记录失败 |

---

## 总结

RAGFlow 的文档删除流程采用了**多层清理**的设计:

1. **任务取消** - 避免资源浪费
2. **索引清理** - 保证搜索结果准确性
3. **数据库清理** - 使用事务保证一致性
4. **存储清理** - 回收磁盘空间

**关键设计原则**:
- 先清理衍生数据 (ES 索引),再删除源数据 (数据库记录)
- 单个组件失败不中断整个流程
- 优先保证数据库一致性,其他存储最终一致
- 删除操作是永久性的,没有软删除或回收站机制

**注意事项**:
- 删除是不可逆的,建议在 API 层增加二次确认
- 批量删除时注意性能影响
- 定期检查并清理孤立的存储对象和 ES 文档
