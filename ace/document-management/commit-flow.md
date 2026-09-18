# 版本历史与提交记录流程

> **范围**: FileCommit 版本控制系统，工作区文件变更追踪与工件页面历史管理

## 概述

RAGFlow 实现了类 Git 的提交记录系统（FileCommit），支持两种独立的提交域：

1. **工作区文件提交**: 追踪租户文件系统中工作文件的版本历史
2. **工件页面提交**: 记录知识库工件页面（Artifact Page）的编辑历史，用于构建页面、运行记录、AI 编辑记录

两者共享同一张 `file_commit` 表和 `file_commit_item` 表，通过 `folder_id` 隔离：
- 工作区提交：`folder_id` 为实际文件夹 ID
- 工件页面提交：`folder_id` 直接使用 `kb_id`（知识库 ID）

---

## 流程 1: 创建提交

**端点**: 
- `POST /folders/<folder_id>/commits` — 工作区文件提交
- `POST /datasets/<dataset_id>/commits` — 工件页面提交

### 步骤 1: 路径解析与授权

```python
# file_commit_api.py:103-116
def _resolve(entity_id):
    if resolver_type is None:
        # entity_id 即 folder_id，验证文件夹归属与权限
        e, folder = FileService.get_by_id(entity_id)
        if not e or not check_file_team_permission(folder, current_user.id):
            raise ValueError(f"Could not resolve folder '{entity_id}'")
        return entity_id
    
    # 通过 resolver_type (如 "datasets") 解析 entity_id → folder_id
    folder_id = _resolve_folder_id(resolver_type, entity_id)
    if folder_id is None:
        raise ValueError(f"Could not resolve {resolver_type} '{entity_id}' to a folder")
    return folder_id
```

**工件页面特殊处理**:
```python
# file_commit_api.py:64-85
@_register_resolver("datasets")
def _resolve_dataset_folder(dataset_id):
    # 工件页面提交使用 kb_id 作为 folder_id
    if not KnowledgebaseService.accessible(dataset_id, current_user.id):
        return None
    return dataset_id  # 直接返回 dataset_id
```

### 步骤 2: 提交创建

```python
# file_commit_api.py:122-145
@validate_request("message", "files")
async def create_commit(entity_id):
    folder_id = _resolve(entity_id)
    req = await get_request_json()
    
    commit = FileCommitService.create_commit(
        folder_id=folder_id,
        author_id=current_user.id,
        message=req["message"],
        file_changes=req["files"],  # [{"file_id": "...", "operation": "add|modify|delete", "old_hash": "...", "new_hash": "..."}]
    )
    
    return get_json_result(data={
        "id": commit.id,
        "folder_id": commit.folder_id,
        "parent_id": commit.parent_id,  # 父提交 ID，构建提交链
        "message": commit.message,
        "author_id": commit.author_id,
        "file_count": commit.file_count,
        "tree_state": commit.tree_state,  # 快照标识
        "create_time": commit.create_time,
    })
```

**关键字段**:
- `file_changes`: 文件变更列表，每项包含 `file_id`、`operation`（add/modify/delete/rename）、hash 对
- `parent_id`: 链接到前一次提交，形成有向无环图（DAG）
- `tree_state`: 当前提交的文件树快照标识

---

## 流程 2: 列出提交历史

**端点**: 
- `GET /folders/<folder_id>/commits` — 工作区提交历史
- `GET /datasets/<dataset_id>/commits` — 工件页面提交历史

### 步骤 1: 分支判断

```python
# file_commit_api.py:154-178
page = validate_rest_api_page(request.args.get("page", DEFAULT_PAGE))
page_size = validate_rest_api_page_size(request.args.get("page_size", DEFAULT_PAGE_SIZE))
slug = request.args.get("slug") or ""

if slug:
    # 工件页面单页历史：通过 slug (page_type/name) 过滤
    total, items = FileCommitService.list_page_commits(
        tenant_id="",
        kb_id=folder_id,
        slug=slug,
        page=page,
        page_size=page_size,
    )
    return get_json_result(data={"total": total, "page": page, "page_size": page_size, "commits": items})
```

**slug 查询**: 工件页面提交通过 `slug=build/<name>` 或 `slug=run/<session_id>` 筛选特定页面的历史，利用 `FileCommitItem.slug_kwd` 索引加速。

### 步骤 2: 通用列表查询

```python
# file_commit_api.py:180-202
order_by = request.args.get("order_by", "create_time")
desc = request.args.get("desc", "true").lower() != "false"

commits, total = FileCommitService.list_commits(folder_id, page, page_size, order_by, desc)
return get_json_result(data={
    "total": total,
    "page": page,
    "page_size": page_size,
    "commits": [
        {
            "id": c.id,
            "folder_id": c.folder_id,
            "parent_id": c.parent_id,
            "message": c.message,
            "author_id": c.author_id,
            "file_count": c.file_count,
            "create_time": c.create_time,
            "title": getattr(c, "title", None),       # 工件提交扩展字段
            "comments": getattr(c, "comments", None),  # 工件提交扩展字段
        }
        for c in commits
    ],
})
```

**响应字段差异**:
- 工作区提交：`title` 和 `comments` 为 `None`
- 工件提交：携带 `title`（页面标题）和 `comments`（AI 编辑摘要）

---

## 流程 3: 获取提交详情

**端点**: `GET /folders/<folder_id>/commits/<commit_id>` 或 `GET /datasets/<dataset_id>/commits/<commit_id>`

### 步骤 1: 提交类型识别

```python
# file_commit_api.py:213-232
commit = FileCommitService.get_commit(commit_id)
if not commit or commit.folder_id != folder_id:
    return get_data_error_result("Commit not found")

# 通过 title 字段判断是否为工件提交
if getattr(commit, "title", None):
    # 工件提交路径：返回增强响应，包含 content_after（页面内容）
    detail = FileCommitService.get_page_commit_detail(
        tenant_id="",
        kb_id=folder_id,
        commit_id=commit_id,
    )
    return get_json_result(data=detail)
```

**工件提交详情**: 从 Blob 存储（MinIO）读取 `content_after`，返回完整页面快照内容，用于渲染历史版本。

### 步骤 2: 工作区提交详情

```python
# file_commit_api.py:234-256
items = FileCommitService.list_commit_files(commit_id)
return get_json_result(data={
    "id": commit.id,
    "folder_id": commit.folder_id,
    "parent_id": commit.parent_id,
    "message": commit.message,
    "author_id": commit.author_id,
    "file_count": commit.file_count,
    "create_time": commit.create_time,
    "files": [
        {
            "file_id": item.file_id,
            "operation": item.operation,  # add/modify/delete/rename
            "old_hash": item.old_hash,
            "new_hash": item.new_hash,
            "old_name": item.old_name,
            "new_name": item.new_name,
        }
        for item in items
    ],
})
```

**文件变更列表**: 包含本次提交中所有文件的操作类型和哈希变化，用于构建 diff 视图。

---

## 流程 4: 比较两个提交

**端点**: `GET /folders/<folder_id>/commits/diff?from=<commit_id>&to=<commit_id>`

```python
# file_commit_api.py:292-310
from_id = request.args.get("from")
to_id = request.args.get("to")

from_commit = FileCommitService.get_commit(from_id)
to_commit = FileCommitService.get_commit(to_id)

# 验证两个提交都属于同一 folder_id
if from_commit.folder_id != folder_id or to_commit.folder_id != folder_id:
    return get_data_error_result("Commit not found in workspace")

diff = FileCommitService.diff_commits(from_id, to_id)
return get_json_result(data=diff)
```

**差异计算**: 聚合从 `from_id` 到 `to_id` 之间的所有变更，生成累积 diff（类似 `git diff <from>..<to>`）。

---

## 流程 5: 查询未提交变更

**端点**: `GET /folders/<folder_id>/changes`

```python
# file_commit_api.py:313-321
changes = FileCommitService.get_uncommitted_changes(folder_id)
return get_json_result(data=changes)
```

**用途**: 对比工作区当前文件状态与最新提交的快照，返回尚未提交的变更列表（新增/修改/删除的文件），用于"提交前预览"。

---

## 流程 6: 获取提交文件树

**端点**: `GET /folders/<folder_id>/commits/<commit_id>/tree`

```python
# file_commit_api.py:324-337
commit = FileCommitService.get_commit(commit_id)
if not commit or commit.folder_id != folder_id:
    return get_data_error_result("Commit not found")

tree = FileCommitService.get_commit_tree(commit_id)
return get_json_result(data=tree)
```

**文件树快照**: 返回该提交时刻的完整文件树结构（文件夹层级 + 文件列表），用于"浏览历史版本"功能。

---

## 流程 7: 获取提交中文件内容

**端点**: `GET /folders/<folder_id>/commits/<commit_id>/files/<file_id>/content`

**用途**: 读取指定提交中某个文件的历史版本内容，用于"查看历史文件"或"恢复旧版本"。

---

## 流程 8: 文件版本历史查询

**端点**: `GET /workspace-files/<file_id>/versions`

**用途**: 获取单个工作区文件的所有提交历史，返回该文件在不同提交中的版本列表（按时间倒序）。

**使用场景**: "文件属性 → 查看历史版本" 功能，无需遍历整个文件夹的提交记录。

---

## 设计考量

### 1. 双提交域隔离

虽然共享表结构，但两种提交域完全独立：

| 特征 | 工作区提交 | 工件页面提交 |
|------|-----------|-------------|
| `folder_id` | 真实文件夹 ID | 知识库 ID (kb_id) |
| 提交触发 | 用户手动提交 | AI 编辑页面时自动记录 |
| `file_commit_item.slug_kwd` | NULL | `build/<name>` 或 `run/<session_id>` |
| 内容存储 | 文件哈希引用 | Blob 存储完整内容快照 |
| 查询路径 | `/folders/<id>/commits` | `/datasets/<id>/commits?slug=<slug>` |

**冲突避免**: `folder_id` 值域不重叠（文件夹 ID 和知识库 ID 来自不同主键空间），且 `slug_kwd` 索引仅对工件提交非空。

### 2. 类 Git 设计模式

**提交链**: 通过 `parent_id` 构建有向无环图（DAG），支持：
- 线性历史查询（最新提交 → 父提交 → 祖父提交）
- 分支与合并（虽然当前未实现分支 UI，但数据模型支持）

**快照标识**: `tree_state` 字段标识文件树状态，可用于：
- 快速判断两个提交是否相同
- 实现增量备份（相同 `tree_state` 无需重复存储）

### 3. 工件页面提交的特殊性

**自动化记录**: 
- AI 编辑工件页面时，通过 `FileCommitService.record_page_edit()` 自动创建提交
- 无需用户手动触发，每次编辑都留下完整快照

**内容存储策略**:
- `content_after` 存储到 Blob（MinIO bucket），`file_commit_item` 仅保存引用路径
- 支持大页面内容（数十 KB 的 Markdown），避免数据库行膨胀

**slug 索引优化**:
```sql
-- file_commit_item 表索引
CREATE INDEX idx_slug_kwd ON file_commit_item (slug_kwd, create_time DESC);
```
使查询 `GET /datasets/<kb_id>/commits?slug=build/overview` 能直接定位到该页面的所有历史版本。

### 4. 哈希冲突处理

工作区提交使用文件内容哈希（SHA256）作为 `new_hash`：
- 相同内容的文件共享哈希，节省存储
- 哈希冲突极低（2^256 空间），生产环境可忽略

工件提交直接存储 `content_after`，不依赖哈希去重。

---

## 相关流程

- [文件管理流程](./file-folder-flow.md) — 工作区文件的修改会触发未提交变更
- [上传流程](./upload-flow.md) — 上传新文件后可作为 `add` 操作提交
- [删除流程](./delete-flow.md) — 删除文件需提交以记录 `delete` 操作

---

## 性能特征

| 操作 | 时间复杂度 | 数据库查询数 | 备注 |
|------|-----------|-------------|------|
| 创建提交 | O(N) | 1 + N | 1 次插入 file_commit + N 次插入 file_commit_item |
| 列出提交历史 | O(M) | 1 | M = page_size，利用 folder_id 索引 |
| 获取提交详情 | O(1 + N) | 2 | 1 次查 commit + 1 次查关联的 items |
| 比较两个提交 | O(K) | 2 + K | K = from 到 to 之间的提交数，需遍历 DAG |
| 工件页面 slug 查询 | O(M) | 1 | 利用 slug_kwd 索引，无需扫描全表 |

**优化建议**:
1. `parent_id` 可加索引以加速 DAG 遍历（实现 `git log` 风格的历史回溯）
2. `tree_state` 可用于实现客户端缓存（相同快照无需重新拉取）

---

## 代码追溯

| 层次 | 文件 | 关键函数 |
|------|------|---------|
| API | `file_commit_api.py` | `create_commit()`, `list_commits()`, `get_commit()`, `diff_commits()` |
| 路由注册 | `file_commit_api.py` | `_register_commit_routes()` — 动态生成 8 个端点 |
| 解析器 | `file_commit_api.py` | `_resolve_dataset_folder()` — 工件提交路径解析 |
| 数据访问 | `file_commit_service.py` | `create_commit()`, `list_page_commits()`, `get_page_commit_detail()` |
| 模型 | `db_models.py` | `FileCommit`, `FileCommitItem` |
