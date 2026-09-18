# RAGFlow 知识库管理数据表详细说明

本文档详细说明 RAGFlow 项目中与知识库管理相关的 9 张数据库表的结构、字段含义和关系。

数据库模型定义位置：[api/db/db_models.py](file:///d:/work/rag/ragflow/api/db/db_models.py)

---

## 目录

1. [knowledgebase - 知识库表](#1-knowledgebase---知识库表)
2. [document - 文档表](#2-document---文档表)
3. [file - 文件表](#3-file---文件表)
4. [file2document - 文件文档关联表](#4-file2document---文件文档关联表)
5. [file_commit - 文件提交表](#5-file_commit---文件提交表)
6. [file_commit_item - 文件提交项表](#6-file_commit_item---文件提交项表)
7. [connector - 连接器表](#7-connector---连接器表)
8. [connector2kb - 连接器知识库关联表](#8-connector2kb---连接器知识库关联表)
9. [sync_logs - 同步日志表](#9-sync_logs---同步日志表)

---

## 1. knowledgebase - 知识库表

**表名**: `knowledgebase`

**用途**: 存储知识库（数据集）的基本信息和配置

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 知识库唯一标识 |
| **tenant_id** | CharField(32) | 是 | 租户ID |
| **name** | CharField(128) | 是 | 知识库名称 |
| **avatar** | TextField | - | 头像（base64字符串） |
| **language** | CharField(32) | 是 | 语言（English/Chinese），默认根据环境变量LANG判断 |
| **description** | TextField | - | 知识库描述 |
| **permission** | CharField(16) | 是 | 权限（me/team），默认"me" |
| **created_by** | CharField(32) | 是 | 创建者ID |
| **doc_num** | IntegerField | 是 | 文档数量 |
| **token_num** | IntegerField | 是 | Token总数 |
| **chunk_num** | IntegerField | 是 | 切块数量 |

### Embedding 配置

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **embd_id** | EmptyStringCharField(128) | 是 | 默认 Embedding 模型ID |
| **tenant_embd_id** | CharField(32) | 是 | tenant_model 表中的 Embedding 模型ID |

### 检索配置

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **similarity_threshold** | FloatField | 是 | 相似度阈值，默认 0.2 |
| **vector_similarity_weight** | FloatField | 是 | 向量相似度权重，默认 0.3 |

### 解析配置

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **parser_id** | CharField(32) | 是 | 默认解析器ID，默认值 `ParserType.NAIVE.value` |
| **pipeline_id** | CharField(32) | 是 | Pipeline ID（新架构） |
| **parser_config** | JSONField | - | 解析器配置，默认：<br>`{"pages": [[1, 1000000]], "table_context_size": 0, "image_context_size": 0}` |
| **pagerank** | IntegerField | - | PageRank 值，默认 0 |

### 任务跟踪字段（12组任务）

每组任务包含两个字段：`*_task_id` 和 `*_task_finish_at`

| 任务类型 | 字段前缀 | 说明 |
|---------|---------|------|
| Graph RAG | graphrag | 图谱 RAG 任务 |
| RAPTOR | raptor | RAPTOR 任务 |
| Mindmap | mindmap | 思维导图任务 |
| Wiki/Artifact | wiki | 产物编译任务 |
| Skill | skill | 技能生成任务 |
| Structure Graph | structure_graph | 结构图谱合并任务 |
| Structure Mindmap | structure_mindmap | 结构思维导图合并任务 |
| Timeline | timeline | 时间线合并任务 |
| Session Graph | session_graph | 会话图谱合并任务 |
| Session Essence | session_essence | 会话精华合并任务 |
| Structure (All) | structure | 结构合并全量任务 |

示例：
```python
graphrag_task_id = CharField(max_length=32, null=True, help_text="Graph RAG task ID", index=True)
graphrag_task_finish_at = DateTimeField(null=True)
```

### 通用字段

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **status** | CharField(1) | 是 | 状态：0=废弃，1=有效，默认"1" |
| **create_time** | BigIntegerField | 是 | 创建时间戳（继承自 BaseModel） |
| **create_date** | DateTimeField | 是 | 创建日期（继承自 BaseModel） |
| **update_time** | BigIntegerField | 是 | 更新时间戳（继承自 BaseModel） |
| **update_date** | DateTimeField | 是 | 更新日期（继承自 BaseModel） |

---

## 2. document - 文档表

**表名**: `document`

**用途**: 存储知识库中的文档信息

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 文档唯一标识 |
| **kb_id** | CharField(256) | 是 | 所属知识库ID |
| **name** | CharField(255) | 是 | 文件名 |
| **thumbnail** | TextField | - | 缩略图（base64字符串） |
| **type** | CharField(32) | 是 | 文件扩展名 |
| **suffix** | EmptyStringCharField(32) | 是 | 真实文件扩展名后缀 |
| **source_type** | CharField(128) | 是 | 来源类型，默认"local" |
| **location** | CharField(255) | 是 | 存储位置 |
| **size** | BigIntegerField | 是 | 文件大小（字节） |
| **created_by** | CharField(32) | 是 | 创建者ID |

### 解析配置

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **parser_id** | CharField(32) | 是 | 使用的解析器ID |
| **pipeline_id** | CharField(32) | 是 | Pipeline ID |
| **parser_config** | JSONField | - | 解析器配置，默认同 knowledgebase |

### 处理状态

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **token_num** | IntegerField | 是 | Token 数量 |
| **chunk_num** | IntegerField | 是 | 切块数量 |
| **progress** | FloatField | 是 | 处理进度（0.0-1.0） |
| **progress_msg** | TextField | - | 处理消息，默认空字符串 |
| **process_begin_at** | DateTimeField | 是 | 处理开始时间 |
| **process_duration** | FloatField | - | 处理耗时（秒） |
| **content_hash** | CharField(32) | 是 | 内容哈希（xxhash128），用于变更检测 |
| **run** | CharField(1) | 是 | 运行控制：0=停止，1=运行，2=取消，默认"0" |
| **status** | CharField(1) | 是 | 状态：0=废弃，1=有效，默认"1" |

---

## 3. file - 文件表

**表名**: `file`

**用途**: 存储文件和文件夹的树形结构信息

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 文件/文件夹唯一标识 |
| **parent_id** | CharField(32) | 是 | 父文件夹ID |
| **tenant_id** | CharField(32) | 是 | 租户ID |
| **created_by** | CharField(32) | 是 | 创建者ID |
| **name** | CharField(255) | 是 | 文件名或文件夹名 |
| **location** | CharField(255) | 是 | 存储位置 |
| **size** | BigIntegerField | 是 | 文件大小（字节） |
| **type** | CharField(32) | 是 | 文件扩展名（文件夹为特殊值） |
| **source_type** | EmptyStringCharField(128) | 是 | 来源类型，默认空字符串 |

### 特殊说明

- 文件夹也作为记录存储在此表中
- 通过 `parent_id` 构建树形结构
- 根文件夹的 `parent_id` 通常为特殊值

---

## 4. file2document - 文件文档关联表

**表名**: `file2document`

**用途**: 关联文件表和文档表（多对多关系）

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 关联记录唯一标识 |
| **file_id** | CharField(32) | 是 | 文件ID（file 表） |
| **document_id** | CharField(32) | 是 | 文档ID（document 表） |

### 使用场景

- 一个文件可以生成多个文档
- 一个文档可以来自多个文件片段
- 支持文件拆分和合并场景

---

## 5. file_commit - 文件提交表

**表名**: `file_commit`

**用途**: 记录文件提交历史（类似 Git Commit）

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 提交唯一标识 |
| **folder_id** | CharField(32) | 是 | 工作区文件夹ID |
| **parent_id** | CharField(32) | 是 | 父提交ID（构建提交链） |
| **author_id** | CharField(32) | 是 | 提交作者ID |
| **message** | CharField(512) | - | 提交消息，默认空字符串 |
| **file_count** | IntegerField | - | 本次提交包含的文件数量 |
| **tree_state** | LongTextField | - | 完整文件夹树的JSON快照 |

### Artifact 提交扩展字段

仅用于通过 `FileCommitService.record_page_edit` 记录的产物页面保存：

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **title** | CharField(255) | - | 提交标题（产物页面编辑） |
| **comments** | TextField | - | 提交正文/描述（产物页面编辑） |

### 版本控制特性

- 支持链式提交历史（通过 `parent_id`）
- 记录完整文件树快照
- 支持工作区文件和产物页面两种提交模式

---

## 6. file_commit_item - 文件提交项表

**表名**: `file_commit_item`

**用途**: 记录每次提交中具体文件的变更详情

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 提交项唯一标识 |
| **commit_id** | CharField(32) | 是 | 所属提交ID |
| **file_id** | CharField(32) | 是 | 文件ID |
| **operation** | CharField(16) | 是 | 操作类型：add/modify/delete/rename |

### 变更跟踪字段

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **old_hash** | CharField(64) | 是 | 旧内容哈希 |
| **new_hash** | CharField(64) | - | 新内容哈希 |
| **old_location** | CharField(255) | - | 旧存储位置 |
| **new_location** | CharField(255) | - | 新存储位置 |
| **old_name** | CharField(255) | - | 旧文件名（用于 rename） |
| **new_name** | CharField(255) | - | 新文件名（用于 rename） |

### Artifact 提交扩展字段

仅用于产物页面保存：

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **diff** | LongTextField | - | 预计算的统一差异（unified diff） |
| **content_after_storage** | CharField(16) | 是 | 保存后内容存储位置：minio/es |
| **content_after_location** | CharField(512) | - | 保存后内容的存储键/ID |
| **slug_kwd** | CharField(512) | 是 | 产物页面 slug（格式：`<page_type>/<name>`） |
| **page_type_kwd** | CharField(32) | 是 | 产物页面类型 |

### 唯一索引

复合唯一索引：`(commit_id, file_id)`

---

## 7. connector - 连接器表

**表名**: `connector`

**用途**: 存储外部数据源连接器配置

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 连接器唯一标识 |
| **tenant_id** | CharField(32) | 是 | 租户ID |
| **name** | CharField(128) | - | 连接器名称 |
| **source** | CharField(128) | 是 | 数据源类型（如：github, gitlab, confluence等） |
| **input_type** | CharField(128) | 是 | 输入类型：poll/event/... |
| **config** | JSONField | - | 连接器配置（JSON对象） |

### 调度配置

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **refresh_freq** | IntegerField | - | 刷新频率（秒），0表示不自动刷新 |
| **prune_freq** | IntegerField | - | 清理频率（秒），0表示不清理 |
| **timeout_secs** | IntegerField | - | 超时时间（秒），默认 3600 |
| **indexing_start** | DateTimeField | 是 | 索引开始时间 |
| **status** | CharField(16) | 是 | 状态：schedule（调度中），默认"schedule" |

### 支持的数据源示例

- GitHub
- GitLab
- Confluence
- Google Drive
- Notion
- Slack
- 等等...

---

## 8. connector2kb - 连接器知识库关联表

**表名**: `connector2kb`

**用途**: 关联连接器和知识库（多对多关系）

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 关联记录唯一标识 |
| **connector_id** | CharField(32) | 是 | 连接器ID（connector 表） |
| **kb_id** | CharField(32) | 是 | 知识库ID（knowledgebase 表） |
| **auto_parse** | CharField(1) | - | 是否自动解析：0=否，1=是，默认"1" |

### 使用场景

- 一个连接器可以同步到多个知识库
- 一个知识库可以从多个连接器获取数据
- 控制是否自动解析新同步的文档

---

## 9. sync_logs - 同步日志表

**表名**: `sync_logs`

**用途**: 记录连接器同步任务的执行日志

### 字段说明

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **id** | CharField(32) | 主键 | 日志唯一标识 |
| **connector_id** | CharField(32) | 是 | 连接器ID |
| **kb_id** | CharField(32) | 是 | 知识库ID |
| **task_type** | CharField(32) | 是 | 任务类型，默认"sync" |
| **status** | CharField(128) | 是 | 处理状态 |
| **from_beginning** | CharField(1) | - | 是否从头开始：0=否，1=是，默认"0" |

### 统计字段

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **new_docs_indexed** | IntegerField | - | 新索引文档数，默认 0 |
| **total_docs_indexed** | IntegerField | - | 总索引文档数，默认 0 |
| **docs_removed_from_index** | IntegerField | - | 从索引中删除的文档数，默认 0 |

### 错误信息

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **error_msg** | EmptyStringTextField | - | 错误消息，默认空字符串 |
| **error_count** | IntegerField | - | 错误计数，默认 0 |
| **full_exception_trace** | EmptyStringTextField | - | 完整异常堆栈，默认空字符串 |

### 时间跟踪

| 字段名 | 类型 | 索引 | 说明 |
|-------|------|------|------|
| **time_started** | DateTimeField | 是 | 任务开始时间 |
| **poll_range_start** | DateTimeTzField | 是 | 轮询范围开始时间（带时区） |
| **poll_range_end** | DateTimeTzField | 是 | 轮询范围结束时间（带时区） |

### 特殊说明

- `DateTimeTzField` 是自定义字段类型，存储带时区的 ISO 格式日期时间
- GaussDB 兼容性：空字符串字段使用 `EmptyStringTextField` 处理 NULL 映射

---

## 表关系图

```
[tenant]
   |
   ├── [knowledgebase] ──┬── [document]
   │                     │      │
   │                     │      └── [file2document] ── [file]
   │                     │                               │
   │                     │                               ├── [file_commit]
   │                     │                               │      │
   │                     │                               │      └── [file_commit_item]
   │                     │
   │                     └── [connector2kb] ── [connector]
   │                                               │
   │                                               └── [sync_logs]
   │
   └── [connector] ──┬── [connector2kb]
                     └── [sync_logs]
```

### 关系说明

1. **tenant → knowledgebase**: 一对多，一个租户可以拥有多个知识库
2. **knowledgebase → document**: 一对多，一个知识库包含多个文档
3. **document ↔ file**: 多对多，通过 `file2document` 关联
4. **file → file_commit**: 一对多，文件有多个提交历史
5. **file_commit → file_commit_item**: 一对多，每次提交包含多个文件变更
6. **knowledgebase ↔ connector**: 多对多，通过 `connector2kb` 关联
7. **connector → sync_logs**: 一对多，连接器有多个同步日志

---

## 数据流程

### 1. 文档上传流程

```
用户上传文件
    ↓
创建 file 记录（parent_id 指向文件夹）
    ↓
创建 document 记录（kb_id 指向知识库）
    ↓
创建 file2document 关联
    ↓
文档解析任务开始（更新 document.progress）
    ↓
完成解析（更新 document.token_num, chunk_num）
```

### 2. 连接器同步流程

```
创建 connector 配置
    ↓
创建 connector2kb 关联（绑定到知识库）
    ↓
定时任务触发同步（根据 refresh_freq）
    ↓
创建 sync_logs 记录（status="processing"）
    ↓
从外部源拉取数据
    ↓
为每个文件创建 file 和 document
    ↓
更新 sync_logs 统计信息
    ↓
完成同步（status="success"/"failed"）
```

### 3. 版本控制流程

```
用户修改文件
    ↓
创建 file_commit 记录（parent_id 指向上一次提交）
    ↓
为每个变更的文件创建 file_commit_item
    ↓
记录变更类型（add/modify/delete/rename）
    ↓
存储内容差异和哈希
```

---

## 技术细节

### 1. GaussDB 兼容性

多个字段使用了特殊的字段类型来处理 GaussDB 的空字符串语义：

- `EmptyStringCharField`: 映射 GaussDB 的 NULL 为应用层空字符串
- `EmptyStringTextField`: 同上，用于 TEXT 类型
- `EmptyStringLongTextField`: 同上，用于 LONGTEXT 类型

### 2. JSON 字段默认值

多个 JSON 字段有复杂的默认值：

```python
parser_config = JSONField(null=False, default={
    "pages": [[1, 1000000]],
    "table_context_size": 0,
    "image_context_size": 0
})
```

### 3. 位标志字段

某些字段使用位标志存储多个布尔值（参见 memory 表的 memory_type）

### 4. 时间字段

- `create_time/update_time`: Unix 时间戳（BigIntegerField）
- `create_date/update_date`: 日期时间对象（DateTimeField）
- 两者同时存储，便于不同场景使用

---

## 常用查询示例

### 查询知识库的所有文档

```python
from api.db.db_models import Knowledgebase, Document

kb = Knowledgebase.get_by_id("kb_id_here")
documents = Document.select().where(Document.kb_id == kb.id)
```

### 查询文档的处理进度

```python
document = Document.get_by_id("doc_id_here")
if document.progress < 1.0:
    print(f"处理中: {document.progress * 100:.2f}%")
    print(f"消息: {document.progress_msg}")
```

### 查询连接器的同步历史

```python
from api.db.db_models import Connector, SyncLogs

connector = Connector.get_by_id("connector_id_here")
logs = SyncLogs.select().where(
    SyncLogs.connector_id == connector.id
).order_by(SyncLogs.time_started.desc()).limit(10)

for log in logs:
    print(f"状态: {log.status}, 新增: {log.new_docs_indexed}, 错误: {log.error_count}")
```

### 查询文件的提交历史

```python
from api.db.db_models import File, FileCommit, FileCommitItem

file = File.get_by_id("file_id_here")
commit_items = FileCommitItem.select().where(
    FileCommitItem.file_id == file.id
).order_by(FileCommitItem.create_date.desc())

for item in commit_items:
    commit = FileCommit.get_by_id(item.commit_id)
    print(f"提交: {commit.message}, 操作: {item.operation}")
```

---

## 维护说明

### 索引策略

- 所有外键字段都有索引
- 常用查询字段（如 name, status）有索引
- 时间字段有索引，便于时间范围查询

### 数据清理

- `status='0'` 的记录表示逻辑删除，需要定期物理清理
- `sync_logs` 表会不断增长，建议定期归档历史数据

### 性能优化

- `document` 表的 `progress` 字段频繁更新，考虑批量更新
- `chunk_num` 和 `token_num` 可以通过聚合查询计算，无需每次更新

---

## 相关文件

- 模型定义：[api/db/db_models.py](file:///d:/work/rag/ragflow/api/db/db_models.py)
- 服务层：
  - `api/db/services/knowledgebase_service.py`
  - `api/db/services/document_service.py`
  - `api/db/services/file_service.py`
  - `api/db/services/connector_service.py`

---

**文档版本**: 1.0  
**最后更新**: 2026-09-18  
**维护者**: RAGFlow Team
