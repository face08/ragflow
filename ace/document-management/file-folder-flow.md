# 文件夹与文件管理流程

## 概述

RAGFlow 维护一套独立的虚拟文件系统（File 表），用于组织租户的文档资源。该系统支持文件夹层级、文件上传、移动、重命名、删除，以及通过 `File2Document` 中间表将文件关联到知识库文档。

**核心特性**:
- 租户级根文件夹自动创建（`tenant_id` 作为根 `id`）
- 虚拟路径与物理存储分离（`location` 字段）
- 文件与文档多对一映射（同一文件可链接到多个知识库）
- 存储碰撞自动重命名（追加下划线直到唯一）
- 技能（Skill）文件夹与索引联动删除

---

## 流程 1: 文件上传

**端点**: `POST /files`（`multipart/form-data`）

**参数**:
- `parent_id` (可选) — 目标文件夹 ID，缺省时上传到根文件夹
- `file` (必需) — 一个或多个文件对象

### 执行步骤

**步骤 1: 父文件夹解析**

```python
if not pf_id:
    root_folder = await thread_pool_exec(FileService.get_root_folder, tenant_id)
    pf_id = root_folder["id"]

e, pf_folder = await thread_pool_exec(FileService.get_by_id, pf_id)
if not e:
    return False, "Can't find this folder!"
```

- 未指定 `parent_id` 时自动获取租户根文件夹
- `get_root_folder` 内部若根不存在会自动创建（`id` 即 `tenant_id`）

**步骤 2: 路径层级创建**

```python
if not file_obj.filename:
    file_obj_names = [pf_folder.name, file_obj.filename]
else:
    full_path = "/" + file_obj.filename
    file_obj_names = full_path.split("/")
file_len = len(file_obj_names)

file_id_list = await thread_pool_exec(FileService.get_id_list_by_id, pf_id, file_obj_names, 1, [pf_id])
len_id_list = len(file_id_list)

if file_len != len_id_list:
    last_folder = await thread_pool_exec(FileService.create_folder, file, file_id_list[len_id_list - 1], ...)
```

- 上传时可携带路径（如 `folder1/folder2/file.pdf`），系统会自动创建中间文件夹
- `get_id_list_by_id` 按路径数组逐级查找已存在的节点
- 缺失的中间文件夹通过 `create_folder` 递归创建

**步骤 3: 存储碰撞处理**

```python
filetype = filename_type(file_obj_names[file_len - 1])
location = file_obj_names[file_len - 1]
while await thread_pool_exec(settings.STORAGE_IMPL.obj_exist, last_folder.id, location):
    location += "_"
blob = await thread_pool_exec(file_obj.read)
```

- 虚拟文件名（`name`）与物理存储路径（`location`）分离
- 存储键格式为 `(bucket=last_folder.id, key=location)`
- 碰撞时追加下划线直到存储对象不存在（循环检测）

**步骤 4: File 记录创建**

```python
filename = await thread_pool_exec(duplicate_name, FileService.query, name=file_obj_names[file_len - 1], parent_id=last_folder.id)
await thread_pool_exec(settings.STORAGE_IMPL.put, last_folder.id, location, blob)
file_data = {
    "id": get_uuid(),
    "parent_id": last_folder.id,
    "tenant_id": tenant_id,
    "created_by": tenant_id,
    "type": filetype,
    "name": filename,
    "location": location,
    "size": len(blob),
}
inserted = await thread_pool_exec(FileService.insert, file_data)
```

- `duplicate_name` 对虚拟文件名做冲突处理（追加 `_1`, `_2` 等）
- 先写入存储再插入数据库记录
- `type` 字段通过 `filename_type` 识别（`FileType.FOLDER`, `FileType.PDF`, `FileType.DOCX` 等）

**限额检查**: 上传前会检查租户文档数量上限 `MAX_FILE_NUM_PER_USER`（环境变量，0 表示无限制）。

---

## 流程 2: 文件夹创建

**端点**: `POST /files`（JSON body）

**参数**:
- `name` (必需) — 文件夹名称
- `parent_id` (可选) — 父文件夹 ID
- `type` (可选) — 文件类型（默认 `FOLDER`）

### 执行步骤

```python
async def create_folder(tenant_id: str, name: str, parent_id: str = None, file_type: str = None):
    if not parent_id:
        root_folder = await thread_pool_exec(FileService.get_root_folder, tenant_id)
        parent_id = root_folder["id"]

    success, pf_folder = await thread_pool_exec(FileService.get_by_id, parent_id)
    if not success:
        return False, "Parent folder not found!"

    def _create_folder_sync():
        folder_name = duplicate_name(FileService.query, name=name, parent_id=parent_id)
        folder_data = {
            "id": get_uuid(),
            "parent_id": parent_id,
            "tenant_id": tenant_id,
            "created_by": tenant_id,
            "type": file_type or FileType.FOLDER,
            "name": folder_name,
            "location": "",
            "size": 0,
        }
        return FileService.insert(folder_data)

    created_folder = await thread_pool_exec(_create_folder_sync)
    return True, created_folder.to_json()
```

- 文件夹不占用物理存储，`location` 为空字符串，`size` 为 0
- 虚拟文件名同样通过 `duplicate_name` 避免同级冲突

---

## 流程 3: 文件列表查询

**端点**: `GET /files`

**参数**:
- `parent_id` (可选) — 文件夹 ID，缺省时列出根文件夹
- `keywords` (可选) — 名称关键词过滤
- `page`, `page_size` — 分页参数
- `orderby`, `desc` — 排序字段与方向

### 执行步骤

```python
def list_files(tenant_id: str, args: dict):
    parent_id = args.get("parent_id")
    if not parent_id:
        root_folder = FileService.get_root_folder(tenant_id)
        parent_id = root_folder["id"]

    files, total = FileService.get_by_pf_id(
        parent_id,
        tenant_id,
        args.get("page", 1),
        args.get("page_size", 15),
        args.get("orderby", "create_time"),
        args.get("desc", True),
        args.get("keywords", ""),
    )

    return True, {"total": total, "files": [f.to_json() for f in files]}
```

- `get_by_pf_id` 内部构造 Peewee 查询，支持 `LIKE` 关键词匹配与排序
- 返回当前层级文件/文件夹，不递归子文件夹

---

## 流程 4: 文件移动与重命名

**端点**: `POST /files/move`

**参数**:
- `src_file_ids` (必需) — 源文件 ID 列表
- `dest_file_id` (可选) — 目标文件夹 ID
- `new_name` (可选) — 新文件名（仅单文件重命名时有效）

**语义规则** (Linux `mv` 风格):
- 仅 `dest_file_id` — 批量移动到新文件夹，文件名不变
- 仅 `new_name` — 单文件原地重命名，无存储操作
- 两者都提供 — 移动并重命名
- 两者都缺失 — 拒绝请求

### 执行步骤

**步骤 1: 批量存在性与团队权限校验**

```python
files = FileService.get_by_ids(src_file_ids)
if not files:
    return False, "Source files not found!"

files_dict = {f.id: f for f in files}
for file_id in src_file_ids:
    file = files_dict.get(file_id)
    if not file:
        return False, "File or folder not found!"
    if not file.tenant_id:
        return False, "Tenant not found!"
    if not check_file_team_permission(file, uid):
        return False, "no authorization"
```

- 一次 `IN` 查询取回所有源文件，避免逐个查库
- 逐个校验团队权限，任一失败即整体拒绝（无部分成功）

**步骤 2: 新名称合法性校验**

```python
if new_name:
    file = files_dict[src_file_ids[0]]
    if "/" in new_name:
        return False, 'Name cannot contain "/"'
    if file.type != FileType.FOLDER.value and \
       pathlib.Path(new_name.lower()).suffix != pathlib.Path(file.name.lower()).suffix:
        return False, "The extension of file can't be changed"
    target_parent_id = dest_folder.id if dest_folder else file.parent_id
    for f in FileService.query(name=new_name, parent_id=target_parent_id):
        if f.name == new_name:
            return False, "Duplicated file name in the same folder."
```

- 禁止名称含 `/`（防止绕过层级结构）
- **扩展名不可变更** — 非文件夹类型重命名必须保持后缀一致，因为后缀决定解析器选择
- 目标层级内同名冲突直接拒绝（不像上传那样自动加后缀）

**步骤 3: 循环引用防护**

```python
if dest_folder:
    for file in files:
        if file.type == FileType.FOLDER.value and file.id == dest_folder.id:
            return False, "Cannot move a folder to itself."
    dest_ancestors = FileService.get_all_parent_folders(dest_folder.id)
    dest_ancestor_ids = {f.id for f in dest_ancestors}
    for file in files:
        if file.type == FileType.FOLDER.value and file.id in dest_ancestor_ids:
            return False, "Cannot move a folder into its own subfolder."
```

两道防线：
1. 拒绝把文件夹移动到自身
2. 拒绝把文件夹移入其自身的子孙目录 — 通过遍历目标的祖先链判断

这是必需的，否则 `_move_entry_recursive` 会无限递归。

**步骤 4: 递归移动（文件夹）**

```python
def _move_entry_recursive(source_file_entry, dest_folder_entry, override_name=None):
    if source_file_entry.id == dest_folder_entry.id:
        raise RuntimeError("The folder is already in the target location. ...")
    effective_name = override_name or source_file_entry.name

    if source_file_entry.type == FileType.FOLDER.value:
        existing_folder = FileService.query(name=effective_name, parent_id=dest_folder_entry.id)
        if existing_folder:
            new_folder = existing_folder[0]
        else:
            new_folder = FileService.insert({... "type": FileType.FOLDER.value})

        sub_files = FileService.list_all_files_by_parent_id(source_file_entry.id)
        for sub_file in sub_files:
            _move_entry_recursive(sub_file, new_folder)

        FileService.delete_by_id(source_file_entry.id)
        return
```

文件夹移动采用**重建 + 递归下沉 + 删除原节点**策略：
- 目标已存在同名文件夹时**合并**而非报错（复用 `existing_folder[0]`）
- 递归处理所有子项后删除源文件夹记录
- 文件夹本身无存储对象，只是数据库记录搬迁

**步骤 5: 物理存储移动（文件）**

```python
need_storage_move = dest_folder_entry.id != source_file_entry.parent_id
updates = {}

if need_storage_move:
    new_location = effective_name
    while settings.STORAGE_IMPL.obj_exist(dest_folder_entry.id, new_location):
        new_location += "_"
    try:
        moved = settings.STORAGE_IMPL.move(
            source_file_entry.parent_id, source_file_entry.location,
            dest_folder_entry.id, new_location,
        )
    except Exception as storage_err:
        raise RuntimeError(f"Move file failed at storage layer: {storage_err!s}")
    if moved is False:
        raise RuntimeError("Move file failed at storage layer")
    updates["parent_id"] = dest_folder_entry.id
    updates["location"] = new_location

if override_name:
    updates["name"] = override_name

if updates:
    FileService.update_by_id(source_file_entry.id, updates)

if override_name:
    _rename_linked_documents(source_file_entry.id, override_name)
```

- **`parent_id` 即 bucket** — 跨文件夹移动意味着跨 bucket 迁移存储对象
- 同文件夹内重命名时 `need_storage_move` 为 `False`，只改数据库 `name`，物理 `location` 保持不变
- 存储层异常统一包装为 `RuntimeError`，同时检查返回值 `False`（不同后端错误表达不一致）

**步骤 6: 关联文档同步重命名**

```python
def _rename_linked_documents(file_id, name):
    informs = File2DocumentService.get_by_file_id(file_id)
    for inform in informs:
        if not DocumentService.update_by_id(inform.document_id, {"name": name}):
            raise RuntimeError("Database error (Document rename)!")
```

文件重命名会级联到所有通过 `File2Document` 链接的知识库文档，保持两套视图名称一致。

**纯重命名分支**:

```python
def _move_or_rename_sync():
    if dest_folder:
        for file in files:
            _move_entry_recursive(file, dest_folder, override_name=new_name)
    else:
        file = files[0]
        if not FileService.update_by_id(file.id, {"name": new_name}):
            return False, "Database error (File rename)!"
        _rename_linked_documents(file.id, new_name)
    return True, True
```

无 `dest_folder` 时走轻量路径：单次 `UPDATE` + 级联重命名，完全跳过存储层。

---

## 流程 5: 文件删除

**端点**: `DELETE /files`

**参数**: `ids` (必需) — 待删除文件/文件夹 ID 列表

### 执行步骤

**步骤 1: 逐项独立处理与错误累积**

与移动不同，删除采用**部分成功**语义：单项失败不阻断其余项，最终汇总错误。

```python
success_count = 0
errors = []
for file_id in file_ids:
    try:
        ...
        success_count += 1
    except Exception as e:
        errors.append({"file_id": file_id, "error": str(e)})
```

端点层据此返回不同响应：

```python
if isinstance(result, dict):
    success_count = result.get("success_count", 0)
    errors = result.get("errors", [])
    return get_json_result(
        code=RetCode.DATA_ERROR,
        message=f"Partially deleted {success_count} files with {len(errors)} errors"
                if success_count > 0
                else f"Deleted files failed with {len(errors)} errors",
        data=result,
    )
```

**步骤 2: 权限与源类型过滤**

```python
for file_id in file_ids:
    e, file = FileService.get_by_id(file_id)
    if not e or not file:
        errors.append(f"File or Folder not found: {file_id}")
        continue
    if not file.tenant_id:
        errors.append(f"Tenant not found for file {file_id}")
        continue
    if not check_file_team_permission(file, uid):
        errors.append(f"No authorization for file {file_id}")
        continue
    
    # 跳过知识库根文件与技能空间根
    if file.source_type == FileSource.KNOWLEDGEBASE:
        continue
    if file.source_type == "skill_space":
        continue
```

- 单项权限不足记录错误但继续处理后续项
- **知识库根文件** (`KNOWLEDGEBASE`) 与 **技能空间根** (`skill_space`) 不可直接删除

**步骤 3: 单文件删除（非文件夹）**

```python
def _delete_single_file(file) -> int:
    try:
        if file.location:
            settings.STORAGE_IMPL.rm(file.parent_id, file.location)
    except Exception as e:
        errors.append(f"Failed to remove object {file.parent_id}/{file.location}: {e}")
    
    informs = File2DocumentService.get_by_file_id(file.id)
    for inform in informs:
        doc_id = inform.document_id
        tenant_id = DocumentService.get_tenant_id(doc_id)
        if not DocumentService.remove_document(doc, tenant_id):
            errors.append(f"Failed to remove document {doc_id} for file {file.id}")
    
    File2DocumentService.delete_by_file_id(file.id)
    FileService.delete(file)
    return 1  # 成功删除 1 个文件
```

删除顺序：
1. 物理存储对象（`location` 非空时）
2. 关联文档（通过 `File2Document` 找到所有 `document_id`，逐个调用 `remove_document`）
3. `File2Document` 中间表记录
4. `File` 表记录

**步骤 4: 文件夹递归删除**

```python
def _delete_folder_recursive(folder, tenant_id) -> int:
    deleted = 0
    
    # 技能文件夹特殊处理：先删除索引
    is_skill_folder = False
    current_space_name = None
    
    if folder.source_type != "skill_space":
        ancestor_success, ancestor_folder = _find_ancestor_skill_space(folder.parent_id, tenant_id)
        if ancestor_success:
            is_skill_folder = True
            current_space_name = ancestor_folder.name
    
    if is_skill_folder and current_space_name:
        index_deleted = _delete_skill_index(tenant_id, current_space_name, folder.name, auth_header)
        if not index_deleted:
            errors.append(f"Failed to delete skill index for folder '{folder.name}'. Folder deletion aborted.")
            return deleted  # 索引删除失败时中止，防止孤立索引
    
    # 递归处理子项
    sub_files = FileService.list_all_files_by_parent_id(folder.id)
    for sub_file in sub_files:
        if sub_file.type == FileType.FOLDER.value:
            deleted += _delete_folder_recursive(sub_file, tenant_id)
        else:
            deleted += _delete_single_file(sub_file)
    
    # 删除文件夹自身
    FileService.delete(folder)
    deleted += 1
    
    # 尝试移除 bucket（best-effort）
    if hasattr(settings.STORAGE_IMPL, "remove_bucket"):
        settings.STORAGE_IMPL.remove_bucket(folder.id)
    
    return deleted
```

关键点：
- **技能索引前置删除** — 通过 `_find_ancestor_skill_space` 向上查找所属技能空间，技能文件夹删除前必须先清理索引，失败时中止
- **后序遍历** — 先递归删除所有子项，再删除节点自身
- **bucket 清理** — 文件夹 `id` 作为存储 bucket，删除记录后尝试移除（best-effort，失败不阻断）

---

## 其他端点

**获取父文件夹**: `GET /files/<file_id>/parent`

返回指定文件的直接父文件夹信息。

**获取祖先路径**: `GET /files/<file_id>/ancestors`

返回从根到当前文件的完整路径链（祖先文件夹列表）。

---

## 设计考量

### 1. 虚拟路径与物理存储分离

**优势**:
- 文件可在文件夹间快速移动（仅改 `parent_id`），无需迁移存储
- 同一文件夹内重命名不触及存储层
- 文件夹无物理对象（`location=""`, `size=0`），层级调整零成本

**代价**:
- 跨文件夹移动（即 `parent_id` 变更）必须执行存储 `move` — `parent_id` 即 bucket，跨文件夹即跨 bucket

### 2. File2Document 解耦

单个文件可关联到多个知识库的文档（多对一映射）：
- 文件重命名时需级联更新所有关联文档的 `name` 字段
- 文件删除时需逐个移除关联文档（调用 `DocumentService.remove_document`）
- 文档删除不影响文件本身（仅删中间表记录）

### 3. 扩展名不可变更

非文件夹类型重命名时必须保持后缀一致，原因：
- 后缀通过 `filename_type` 函数映射到解析器（`pdf` → `PdfParser`, `docx` → `DocxParser`）
- 改变后缀会导致已解析文档与当前解析器不匹配，引发重新解析

### 4. 移动与删除的语义差异

| 操作 | 成功语义 | 失败处理 |
|------|----------|----------|
| 移动 | 全成功或全失败 | 任一文件失败即回滚整个请求 |
| 删除 | 部分成功 | 单项失败记录错误，继续处理其余项 |

删除采用宽松策略是因为用户通常期望"尽可能清理"，即使部分项遇到权限或依赖问题。

---

## 性能特征

| 操作 | 时间复杂度 | 存储 I/O | 备注 |
|------|-----------|----------|------|
| 上传文件 | O(path_depth) | 1 次 put | 路径层级需逐层查询/创建 |
| 同文件夹重命名 | O(linked_docs) | 0 | 仅数据库更新 + 级联文档重命名 |
| 跨文件夹移动 | O(1) | 1 次 move | 单文件移动，非文件夹递归 |
| 文件夹递归移动 | O(total_files) | N 次 move | 每个文件独立 move |
| 文件夹递归删除 | O(total_files) | N 次 rm + 1 次 remove_bucket | 技能文件夹需额外调用索引删除 API |

---

## 相关流程

- [上传流程](./upload-flow.md) — 文档初始入库与解析触发
- [删除流程](./delete-flow.md) — 知识库文档删除（与本流程的文件删除不同层）
- [查询流程](./query-flow.md) — 文件列表的过滤与排序实现
- [下载流程](./download-flow.md) — 文件下载的权限校验与流式传输

---

## 代码追溯

```
文件管理端点层
├─ api/apps/restful_apis/file_api.py
│  ├─ POST /files                    → upload_file / create_folder
│  ├─ GET /files                     → list_files
│  ├─ DELETE /files                  → delete_files
│  ├─ POST /files/move               → move_files
│  ├─ GET /files/<file_id>           → download (见 download-flow.md)
│  ├─ GET /files/<file_id>/parent    → get_parent_folder
│  └─ GET /files/<file_id>/ancestors → get_ancestor_folders

服务层
├─ api/apps/services/file_api_service.py
│  ├─ upload_file(tenant_id, pf_id, file_objs)
│  ├─ create_folder(tenant_id, name, parent_id, file_type)
│  ├─ list_files(tenant_id, args)
│  ├─ move_files(uid, src_file_ids, dest_file_id, new_name)
│  │  └─ _move_entry_recursive(source, dest, override_name)  # 递归移动
│  ├─ delete_files(tenant_id, file_ids, auth_header)
│  │  ├─ _delete_single_file(file)                           # 单文件删除
│  │  ├─ _delete_folder_recursive(folder, tenant_id)         # 文件夹递归删除
│  │  ├─ _find_ancestor_skill_space(folder_id, tenant_id)    # 技能空间查找
│  │  └─ _delete_skill_index(tenant_id, space, skill, auth)  # 技能索引删除
│  └─ _rename_linked_documents(file_id, name)                # 级联文档重命名

数据访问层
├─ api/db/services/file_service.py
│  ├─ get_root_folder(tenant_id)                 # 租户根文件夹
│  ├─ get_id_list_by_id(pf_id, names, type, ids) # 路径层级解析
│  ├─ create_folder(...)                         # 文件夹创建
│  ├─ get_by_pf_id(parent_id, tenant_id, ...)    # 列表查询 + 分页排序
│  ├─ get_by_ids(ids)                            # 批量获取
│  ├─ get_all_parent_folders(file_id)            # 祖先链查询
│  ├─ list_all_files_by_parent_id(parent_id)     # 子项列表（无分页）
│  ├─ update_by_id(id, updates)                  # 单记录更新
│  └─ delete_by_id(id) / delete(file)            # 删除记录

存储层
└─ rag/settings.py::STORAGE_IMPL
   ├─ obj_exist(bucket, key) → bool
   ├─ put(bucket, key, data)
   ├─ move(src_bucket, src_key, dst_bucket, dst_key) → bool
   ├─ rm(bucket, key)
   └─ remove_bucket(bucket)  # 可选接口
```

