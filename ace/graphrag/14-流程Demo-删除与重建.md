# GraphRAG 流程 Demo：删除与重建

## 概述

本文档展示 RAGFlow GraphRAG 模块中与**删除和重建**相关的核心流程，包括：
1. **单文档删除**：删除 KB 中的某个文档，触发全局图重建
2. **知识库清空**：清空所有图数据但保留 KB 结构
3. **知识库删除**：完全删除 KB 及其所有 GraphRAG 数据
4. **仅清理缓存**：清理 Redis 缓存以强制重新计算，但保留 DocStore 数据

每个场景包含：
- 触发条件
- 调用链路
- 数据变更
- 存储位置
- 性能考量

---

## 场景 1：单文档删除

### 场景描述

**初始状态**：
- KB `kb_001` 包含 2 个文档：`doc_A`、`doc_B`
- 全局图已构建完成，包含从两个文档抽取的所有实体和关系
- Redis 中存在 phase markers 和 checkpoints

**操作**：用户删除文档 `doc_B`

**预期结果**：
- `doc_B` 的子图数据被删除
- Phase markers 被清除，触发全局图重建
- 全局图重新计算，仅包含 `doc_A` 的实体和关系
- KB 计数器（token_num、chunk_num、doc_num）原子性更新

---

### 阶段 0：触发删除操作

**入口函数**：`api/db/services/document_service.py:delete_document_and_update_kb_counts()`（lines 847-875）

#### Demo 数据示例

**初始状态（删除前）**：

MySQL `documents` 表：
```sql
SELECT id, kb_id, name, token_num, chunk_num FROM documents WHERE kb_id = 'kb_001';

+--------+--------+-----------+-----------+-----------+
| id     | kb_id  | name      | token_num | chunk_num |
+--------+--------+-----------+-----------+-----------+
| doc_A  | kb_001 | doc_A.txt | 3500      | 15        |
| doc_B  | kb_001 | doc_B.txt | 2500      | 10        |
+--------+--------+-----------+-----------+-----------+
```

MySQL `knowledgebases` 表：
```sql
SELECT id, name, doc_num, token_num, chunk_num FROM knowledgebases WHERE id = 'kb_001';

+--------+-------------+---------+-----------+-----------+
| id     | name        | doc_num | token_num | chunk_num |
+--------+-------------+---------+-----------+-----------+
| kb_001 | ML_KB       | 2       | 6000      | 25        |
+--------+-------------+---------+-----------+-----------+
```

**调用示例**：
```python
# 用户在 UI 上点击删除按钮，前端调用 API
# DELETE /api/v1/document/doc_B

# 后端处理
from api.db.services.document_service import DocumentService

doc_id = "doc_B"
deleted = DocumentService.delete_document_and_update_kb_counts(doc_id)
# 返回: True（文档被成功删除）
```

**数据库状态（删除后）**：

MySQL `documents` 表：
```sql
SELECT id, kb_id, name, token_num, chunk_num FROM documents WHERE kb_id = 'kb_001';

+--------+--------+-----------+-----------+-----------+
| id     | kb_id  | name      | token_num | chunk_num |
+--------+--------+-----------+-----------+-----------+
| doc_A  | kb_001 | doc_A.txt | 3500      | 15        |
+--------+--------+-----------+-----------+-----------+
-- doc_B 行已被删除
```

MySQL `knowledgebases` 表：
```sql
SELECT id, name, doc_num, token_num, chunk_num FROM knowledgebases WHERE id = 'kb_001';

+--------+-------------+---------+-----------+-----------+
| id     | name        | doc_num | token_num | chunk_num |
+--------+-------------+---------+-----------+-----------+
| kb_001 | ML_KB       | 1       | 3500      | 15        |
+--------+-------------+---------+-----------+-----------+
-- doc_num: 2→1, token_num: 6000→3500, chunk_num: 25→15
```

**关键实现**：
```python
@classmethod
@retry_deadlock_operation()
@DB.connection_context()
def delete_document_and_update_kb_counts(cls, doc_id) -> bool:
    """原子性删除文档行并更新 KB 计数器
    
    返回 True 如果文档被此次调用删除，False 如果已被并发请求删除（幂等）
    """
    with DB.atomic():  # 数据库事务，保证原子性
        # 1. 加行锁获取文档
        doc = (
            cls.model.select(
                cls.model.id,
                cls.model.kb_id,
                cls.model.token_num,
                cls.model.chunk_num,
            )
            .where(cls.model.id == doc_id)
            .for_update()  # 行锁，防止并发
            .get_or_none()
        )
        if doc is None:
            return False  # 幂等性保证：文档已被删除
        
        # 2. 删除文档行
        deleted = cls.model.delete().where(cls.model.id == doc_id).execute()
        if not deleted:
            return False
        
        # 3. 原子性更新 KB 计数器
        Knowledgebase.update(
            token_num=Knowledgebase.token_num - doc.token_num,
            chunk_num=Knowledgebase.chunk_num - doc.chunk_num,
            doc_num=Knowledgebase.doc_num - 1,
        ).where(Knowledgebase.id == doc.kb_id).execute()
    
    return True
```

**数据变更**：
| 数据项 | 变更前 | 变更后 |
|--------|--------|--------|
| documents 表 | `doc_B` 行存在 | `doc_B` 行被删除 |
| kb_001.doc_num | 2 | 1 |
| kb_001.chunk_num | 50 | 25（假设 doc_B 有 25 chunks） |
| kb_001.token_num | 10000 | 5000（假设 doc_B 有 5000 tokens） |

**存储位置**：MySQL/PostgreSQL `documents` 和 `knowledgebases` 表

**事务保证**：
- 使用 `DB.atomic()` 确保文档删除和计数器更新在同一事务中
- 使用 `.for_update()` 行锁防止并发删除导致的计数器不一致
- 幂等性设计：重复调用返回 False，不会重复扣减计数器

---

### 阶段 1：删除子图数据

**触发时机**：文档删除后，GraphRAG 任务调度器检测到文档缺失

**删除函数**：`rag/graphrag/utils.py:_save_global_graph()`（lines 706-756）

#### Demo 数据示例

**删除前 DocStore 状态**：

Elasticsearch/Infinity index `tenant_001` 中 `kb_001` 的数据：
```json
// knowledge_graph_kwd = "subgraph"
[
  {
    "id": "chunk_subgraph_doc_A_hash123",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "subgraph",
    "source_id": "doc_A",
    "graph_json": "{\"nodes\": [{\"id\": \"机器学习\", ...}, ...], \"edges\": [...]}"
  },
  {
    "id": "chunk_subgraph_doc_B_hash456",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "subgraph",
    "source_id": "doc_B",
    "graph_json": "{\"nodes\": [{\"id\": \"神经网络\", ...}, ...], \"edges\": [...]}"
  }
]

// knowledge_graph_kwd = "entity"（doc_B 的实体示例）
[
  {
    "id": "chunk_entity_kb_001_神经网络",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "entity",
    "name_kwd": "神经网络",
    "description": "模拟人脑神经元结构的机器学习算法，由多层神经元组成",
    "source_id": ["doc_A", "doc_B"],  // 来自两个文档
    "degree": 12
  },
  {
    "id": "chunk_entity_kb_001_Adam",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "entity",
    "name_kwd": "Adam",
    "description": "自适应学习率优化算法，结合动量和RMSProp",
    "source_id": ["doc_B"],  // 仅来自 doc_B
    "degree": 5
  },
  {
    "id": "chunk_entity_kb_001_SGD",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "entity",
    "name_kwd": "SGD",
    "description": "随机梯度下降，最基础的优化算法",
    "source_id": ["doc_B"],  // 仅来自 doc_B
    "degree": 4
  }
  // ... 更多实体
]

// knowledge_graph_kwd = "relation"（doc_B 相关的边示例）
[
  {
    "id": "chunk_graph_kb_001_edge_Adam_学习率",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "relation",
    "src_id": "Adam",
    "tgt_id": "学习率",
    "description": "Adam优化器通过自适应调整学习率",
    "weight": 9.0,
    "source_id": ["doc_B"]
  },
  {
    "id": "chunk_graph_kb_001_edge_SGD_学习率",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "relation",
    "src_id": "SGD",
    "tgt_id": "学习率",
    "description": "SGD使用固定学习率更新参数",
    "weight": 8.0,
    "source_id": ["doc_B"]
  }
  // ... 更多关系
]
```

**删除操作**：
```python
# 1. 删除旧的全局图数据（graph 和 subgraph）
await thread_pool_exec(
    settings.docStoreConn.delete,
    {"knowledge_graph_kwd": ["graph", "subgraph"]},
    search.index_name(tenant_id),
    kb_id
)

# 2. 批量删除被移除的节点（entity）
# 假设 doc_B 独有的实体：["Adam", "SGD", "优化器", "批次大小", "预训练"]
if change.removed_nodes:
    BATCH_SIZE = 100
    sorted_nodes = sorted(change.removed_nodes)
    for i in range(0, len(sorted_nodes), BATCH_SIZE):
        batch = sorted_nodes[i : i + BATCH_SIZE]
        await thread_pool_exec(
            settings.docStoreConn.delete,
            {"knowledge_graph_kwd": ["entity"], "entity_kwd": batch},
            search.index_name(tenant_id),
            kb_id
        )

# 3. 删除被移除的边（relation），带重试机制
async def del_edges(from_node, to_node):
    max_retries = 3
    for attempt in range(max_retries):
        try:
            async with chat_limiter:
                await thread_pool_exec(
                    settings.docStoreConn.delete,
                    {
                        "knowledge_graph_kwd": ["relation"],
                        "from_entity_kwd": from_node,
                        "to_entity_kwd": to_node
                    },
                    search.index_name(tenant_id),
                    kb_id
                )
            return
        except Exception as e:
            if attempt < max_retries - 1:
                wait = 2**attempt
                logging.warning(
                    f"del_edges({from_node}, {to_node}) attempt {attempt + 1} failed: {e}, "
                    f"retrying in {wait}s"
                )
                await asyncio.sleep(wait)
            else:
                raise

# 批量删除所有被移除的边
# 假设 doc_B 相关的边：[("Adam", "学习率"), ("SGD", "学习率"), ("优化器", "Adam"), ...]
if change.removed_edges:
    await asyncio.gather(*[del_edges(u, v) for u, v in change.removed_edges])
```

**删除后 DocStore 状态**：

```json
// knowledge_graph_kwd = "subgraph"（仅剩 doc_A）
[
  {
    "id": "chunk_subgraph_doc_A_hash123",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "subgraph",
    "source_id": "doc_A",
    "graph_json": "{\"nodes\": [{\"id\": \"机器学习\", ...}], \"edges\": [...]}"
  }
  // doc_B 的 subgraph 已删除
]

// knowledge_graph_kwd = "entity"（doc_B 独有实体已删除）
[
  {
    "id": "chunk_entity_kb_001_神经网络",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "entity",
    "name_kwd": "神经网络",
    "description": "模拟人脑神经元结构的机器学习算法，由多层神经元组成",
    "source_id": ["doc_A", "doc_B"],  // 仍包含 doc_B，将在重建时更新
    "degree": 12
  }
  // "Adam", "SGD" 等 doc_B 独有实体已删除
]

// knowledge_graph_kwd = "relation"（doc_B 相关边已删除）
[
  // "Adam → 学习率", "SGD → 学习率" 等边已删除
]
```

**数据变更统计**：
| 数据类型 | knowledge_graph_kwd | 删除前数量 | 删除后数量 | 说明 |
|----------|---------------------|------------|------------|------|
| 子图 | subgraph | 2 | 1 | 删除 doc_B 的子图 |
| 全局图 | graph | 1 | 0 | 旧全局图被删除，等待重建 |
| 实体 | entity | 69 | ~60 | 删除 doc_B 独有的 ~9 个实体 |
| 关系 | relation | 120 | ~105 | 删除 doc_B 相关的 ~15 条边 |
| 社区报告 | community_report | 6 | 0 | 旧报告被删除，等待重建 |

---

### 阶段 2：清除 Phase Markers

**触发时机**：文档删除后，系统需要标记全局图为"未完成"状态

**清除函数**：`rag/graphrag/phase_markers.py:clear_phase_markers()`（lines 77-85）

#### Demo 数据示例

**清除前 Redis 状态**：
```bash
# Redis keys
127.0.0.1:6379> KEYS graphrag:phase:kb_001:*
1) "graphrag:phase:kb_001:resolution_done"
2) "graphrag:phase:kb_001:community_done"

127.0.0.1:6379> GET graphrag:phase:kb_001:resolution_done
"1789710000"  # Unix timestamp，表示实体消解完成时间

127.0.0.1:6379> GET graphrag:phase:kb_001:community_done
"1789710120"  # Unix timestamp，表示社区检测完成时间

127.0.0.1:6379> TTL graphrag:phase:kb_001:resolution_done
604800  # 7 天 TTL
```

**调用示例**：
```python
from rag.graphrag.phase_markers import clear_phase_markers

kb_id = "kb_001"
clear_phase_markers(kb_id)  # 清除所有 phase markers
```

**清除后 Redis 状态**：
```bash
127.0.0.1:6379> KEYS graphrag:phase:kb_001:*
(empty array)  # 所有 phase markers 已被删除

127.0.0.1:6379> GET graphrag:phase:kb_001:resolution_done
(nil)

127.0.0.1:6379> GET graphrag:phase:kb_001:community_done
(nil)
```

**数据变更**：
| Redis Key | 变更前 | 变更后 |
|-----------|--------|--------|
| `graphrag:phase:kb_001:resolution_done` | 存在（值为 1789710000） | 被删除 |
| `graphrag:phase:kb_001:community_done` | 存在（值为 1789710120） | 被删除 |

---

### 阶段 3：清理 Checkpoints

**触发时机**：文档删除后，旧的 checkpoints 已失效

**清理函数**：`rag/graphrag/checkpoints.py:cleanup_checkpoints()`（lines 120-134）

#### Demo 数据示例

**清理前 Redis 状态**：
```bash
# 实体消解 checkpoint 索引
127.0.0.1:6379> SMEMBERS graphrag:checkpoint_index:tenant_001:kb_001:graphrag_checkpoint_resolution
1) "a1b2c3d4e5f6..."  # checkpoint key 1
2) "f6e5d4c3b2a1..."  # checkpoint key 2
3) "123456789abc..."  # checkpoint key 3
4) "def987654321..."  # checkpoint key 4
5) "abc123def456..."  # checkpoint key 5

# 实体消解 checkpoint 数据（示例 1 个）
127.0.0.1:6379> HGETALL graphrag:checkpoint:tenant_001:kb_001:graphrag_checkpoint_resolution:a1b2c3d4e5f6...
1) "pair_key"
2) "BYTEDANCE<|>字节跳动"
3) "result"
4) "{\"merged_name\": \"BYTEDANCE\", \"reason\": \"Same entity, different case\"}"
5) "timestamp"
6) "1789710050"

127.0.0.1:6379> TTL graphrag:checkpoint:tenant_001:kb_001:graphrag_checkpoint_resolution:a1b2c3d4e5f6...
604800  # 7 天 TTL

# 社区检测 checkpoint 索引
127.0.0.1:6379> SMEMBERS graphrag:checkpoint_index:tenant_001:kb_001:graphrag_checkpoint_community
1) "community_0_hash"
2) "community_1_hash"
3) "community_5_hash"
```

**调用示例**：
```python
from rag.graphrag.checkpoints import cleanup_checkpoints, COMMUNITY_CHECKPOINT, RESOLUTION_CHECKPOINT

tenant_id = "tenant_001"
kb_id = "kb_001"

# 清理实体消解 checkpoints
cleaned_res = await cleanup_checkpoints(tenant_id, kb_id, RESOLUTION_CHECKPOINT)
# 返回 True，日志输出：Cleaned up 5 GraphRAG checkpoints type=graphrag_checkpoint_resolution kb=kb_001

# 清理社区检测 checkpoints
cleaned_comm = await cleanup_checkpoints(tenant_id, kb_id, COMMUNITY_CHECKPOINT)
# 返回 True，日志输出：Cleaned up 3 GraphRAG checkpoints type=graphrag_checkpoint_community kb=kb_001
```

**清理后 Redis 状态**：
```bash
# 所有 checkpoint 索引和数据都已被删除
127.0.0.1:6379> SMEMBERS graphrag:checkpoint_index:tenant_001:kb_001:graphrag_checkpoint_resolution
(empty array)

127.0.0.1:6379> SMEMBERS graphrag:checkpoint_index:tenant_001:kb_001:graphrag_checkpoint_community
(empty array)

127.0.0.1:6379> KEYS graphrag:checkpoint:tenant_001:kb_001:*
(empty array)
```

**实现代码**：
```python
async def cleanup_checkpoints(
    tenant_id: str,
    kb_id: str,
    checkpoint_type: str,
    *,
    page_size: int | None = None
) -> bool:
    """清理指定类型的所有 checkpoints"""
    index_key = _checkpoint_index_key(tenant_id, kb_id, checkpoint_type)
    
    try:
        cleaned_count = 0
        checkpoint_keys = _iter_checkpoint_keys(index_key, page_size)
        
        # 遍历所有 checkpoint keys 并删除
        for checkpoint_key in checkpoint_keys:
            checkpoint_key = _decode_redis_value(checkpoint_key)
            REDIS_CONN.delete(
                _checkpoint_data_key(tenant_id, kb_id, checkpoint_type, checkpoint_key)
            )
            cleaned_count += 1
        
        # 删除索引
        REDIS_CONN.delete(index_key)
        
        logging.info(
            "Cleaned up %d GraphRAG checkpoints type=%s kb=%s",
            cleaned_count, checkpoint_type, kb_id
        )
        return True
    except Exception:
        logging.exception(
            "Failed to cleanup GraphRAG checkpoints type=%s kb=%s",
            checkpoint_type, kb_id
        )
        return False
```

**数据变更统计**：

| Redis Key Pattern | 变更前数量 | 变更后数量 | 说明 |
|-------------------|------------|------------|------|
| `graphrag:checkpoint:tenant_001:kb_001:graphrag_checkpoint_resolution:*` | 5 | 0 | 实体消解 checkpoint 数据 |
| `graphrag:checkpoint:tenant_001:kb_001:graphrag_checkpoint_community:*` | 3 | 0 | 社区检测 checkpoint 数据 |
| `graphrag:checkpoint_index:tenant_001:kb_001:graphrag_checkpoint_resolution` | 1（索引集合） | 0 | 实体消解索引 |
| `graphrag:checkpoint_index:tenant_001:kb_001:graphrag_checkpoint_community` | 1（索引集合） | 0 | 社区检测索引 |

**存储位置**：Redis

**性能考量**：
- 使用 `_iter_checkpoint_keys()` 分页遍历，避免一次性加载所有 keys
- 删除操作是同步的，但数量通常不多（< 100）
- checkpoint 内容示例：实体对消解结果、社区报告哈希等

---

### 阶段 4：全局图重建

**触发时机**：Phase markers 被清除后，下次 GraphRAG 任务检测到需要重建

**重建入口**：`rag/graphrag/general/index.py:run_graphrag_for_kb()`（lines 150-300）

#### Demo 数据示例

**步骤 1：检测 phase markers 缺失**

```python
from rag.graphrag.phase_markers import has_phase_marker, PHASE_RESOLUTION, PHASE_COMMUNITY

kb_id = "kb_001"
need_resolution = not has_phase_marker(kb_id, PHASE_RESOLUTION)  # True，因为已被清除
need_community = not has_phase_marker(kb_id, PHASE_COMMUNITY)    # True，因为已被清除

print(f"需要实体消解: {need_resolution}")  # 输出: 需要实体消解: True
print(f"需要社区检测: {need_community}")  # 输出: 需要社区检测: True
```

**步骤 2：重新加载所有子图**（仅剩 `doc_A` 的子图）

```python
# 从 DocStore 加载所有 knowledge_graph_kwd = "subgraph" 的数据
tenant_id = "tenant_001"
kb_id = "kb_001"

subgraphs = await load_subgraphs_from_docstore(tenant_id, kb_id)
# 结果：仅包含 doc_A 的子图，doc_B 的子图已在阶段 1 被删除

print(f"加载到 {len(subgraphs)} 个子图")  # 输出: 加载到 1 个子图
```

**加载到的子图数据**（doc_A 的子图，简化示例）：
```json
{
  "directed": false,
  "multigraph": false,
  "graph": {},
  "nodes": [
    {"id": "BYTEDANCE", "entity_type": "ORGANIZATION", "description": "字节跳动公司<SEP>总部位于北京", "source_id": ["doc_A_chunk_001", "doc_A_chunk_003"]},
    {"id": "TIKTOK", "entity_type": "PRODUCT", "description": "短视频应用", "source_id": ["doc_A_chunk_005"]},
    {"id": "DOUYIN", "entity_type": "PRODUCT", "description": "国内版抖音", "source_id": ["doc_A_chunk_006"]},
    {"id": "AI LAB", "entity_type": "ORGANIZATION", "description": "人工智能实验室", "source_id": ["doc_A_chunk_010"]}
  ],
  "edges": [
    {"source": "BYTEDANCE", "target": "TIKTOK", "weight": 8.5, "keywords": "开发,推出", "description": "字节跳动开发了 TikTok", "source_id": ["doc_A_chunk_005"]},
    {"source": "BYTEDANCE", "target": "DOUYIN", "weight": 9.0, "keywords": "开发,运营", "description": "字节跳动运营抖音", "source_id": ["doc_A_chunk_006"]},
    {"source": "BYTEDANCE", "target": "AI LAB", "weight": 7.5, "keywords": "设立,研究", "description": "字节跳动设立 AI 实验室", "source_id": ["doc_A_chunk_010"]}
  ]
}
```

**步骤 3：合并子图**

```python
import networkx as nx
from rag.graphrag.utils import graph_merge

# 将所有子图合并为全局图
global_graph = nx.Graph()
for subgraph_json in subgraphs:
    subgraph = nx.node_link_graph(json.loads(subgraph_json))
    global_graph = graph_merge(global_graph, subgraph)

print(f"全局图节点数: {global_graph.number_of_nodes()}")  # 输出: 全局图节点数: 52
print(f"全局图边数: {global_graph.number_of_edges()}")    # 输出: 全局图边数: 98
```

**合并后的全局图统计**：
- 节点数：52（仅 doc_A 的实体，doc_B 的实体已被删除）
- 边数：98（仅 doc_A 的关系）
- 对比删除前（doc_A + doc_B）：节点 69 → 52，边 120 → 98

**步骤 4：实体消解**（因为 PHASE_RESOLUTION 被清除）

```python
from rag.graphrag.entity_resolution import EntityResolution

entity_resolution = EntityResolution(extractor_config)
resolved_graph = await entity_resolution.resolve(global_graph)

print(f"消解后节点数: {resolved_graph.number_of_nodes()}")  # 输出: 消解后节点数: 48
print(f"消解合并了 {52 - 48} 个重复实体")  # 输出: 消解合并了 4 个重复实体
```

**消解示例**（rapidfuzz 预过滤 + LLM 判断）：
```
合并对: "BYTEDANCE" ← "字节跳动"（相似度 0.87，LLM 判断为同一实体）
合并对: "AI LAB" ← "Artificial Intelligence Lab"（相似度 0.92，LLM 判断为同一实体）
合并对: "TIKTOK" ← "TikTok"（相似度 1.0，完全匹配）
合并对: "DOUYIN" ← "抖音"（相似度 0.88，LLM 判断为同一实体）
```

**步骤 5：社区检测**（因为 PHASE_COMMUNITY 被清除）

```python
from rag.graphrag.community_reports_extractor import CommunityReportsExtractor

community_reports_extractor = CommunityReportsExtractor(extractor_config)
communities = await community_reports_extractor.extract(resolved_graph)

print(f"检测到 {len(communities)} 个社区")  # 输出: 检测到 4 个社区
```

**检测到的社区结构**（Leiden 算法）：
```json
[
  {
    "community_id": 0,
    "level": 0,
    "title": "字节跳动核心产品",
    "summary": "字节跳动旗下的核心产品矩阵，包括 TikTok、抖音等短视频平台",
    "rating": 8.5,
    "rating_explanation": "该社区代表字节跳动的主要业务线",
    "findings": ["TikTok 是全球化产品", "抖音是国内市场主力"],
    "entities": ["BYTEDANCE", "TIKTOK", "DOUYIN"],
    "node_count": 12,
    "edge_count": 18
  },
  {
    "community_id": 2,
    "level": 0,
    "title": "内容创作生态",
    "summary": "基于短视频的内容创作者生态系统",
    "rating": 7.2,
    "entities": ["CREATOR", "CONTENT", "VIDEO EDITING"],
    "node_count": 11,
    "edge_count": 15
  },
  {
    "community_id": 3,
    "level": 0,
    "title": "技术基础设施",
    "summary": "支撑产品的底层技术架构",
    "rating": 6.8,
    "entities": ["CLOUD COMPUTING", "DATA CENTER", "CDN"],
    "node_count": 10,
    "edge_count": 12
  }
]
```

**对比删除前社区数量**：6 个社区（doc_A + doc_B）→ 4 个社区（仅 doc_A）

**步骤 6：保存新的全局图**

```python
from rag.graphrag.general.index import _save_global_graph

await _save_global_graph(
    tenant_id, kb_id, resolved_graph, communities, change
)

# 设置 phase markers
from rag.graphrag.phase_markers import set_phase_marker, PHASE_RESOLUTION, PHASE_COMMUNITY

set_phase_marker(kb_id, PHASE_RESOLUTION)   # Redis: graphrag:phase:kb_001:resolution_done
set_phase_marker(kb_id, PHASE_COMMUNITY)    # Redis: graphrag:phase:kb_001:community_done

print("全局图重建完成")
```

**保存的数据**（存入 DocStore）：

| 数据类型 | knowledge_graph_kwd | 数量 | 示例 ID | 说明 |
|----------|---------------------|------|---------|------|
| 全局图 | graph | 1 | `graph_{kb_001}` | 包含 48 个节点、92 条边的合并图 |
| 实体 | entity | 48 | `entity_{BYTEDANCE}` | doc_A 的实体，经过消解 |
| 关系 | relation | 92 | `relation_{BYTEDANCE}__{TIKTOK}` | doc_A 的关系，权重已合并 |
| 社区报告 | community_report | 4 | `community_0_report` | 新检测到的 4 个社区 |

**存储位置**：Elasticsearch/Infinity index `{tenant_id}_knowledge_graph_kwd`

**性能统计示例**：
```json
{
  "stage": "delete_and_rebuild",
  "kb_id": "kb_001",
  "deleted_doc_id": "doc_B",
  "rebuild_time_seconds": 45.3,
  "phases": {
    "subgraph_loading": 2.1,
    "graph_merging": 1.5,
    "entity_resolution": 25.7,
    "community_detection": 12.0,
    "graph_saving": 4.0
  },
  "metrics": {
    "remaining_subgraphs": 1,
    "global_nodes_before_resolution": 52,
    "global_nodes_after_resolution": 48,
    "global_edges": 92,
    "communities": 4,
    "merged_entities": 4
  },
  "comparison": {
    "before_deletion": {
      "nodes": 69,
      "edges": 120,
      "communities": 6
    },
    "after_deletion": {
      "nodes": 48,
      "edges": 92,
      "communities": 4
    }
  }
}
```

---

### 完整调用链

```
用户删除 doc_B
    ↓
DocumentService.delete_document_and_update_kb_counts(doc_B)
    ├─ [MySQL] DELETE FROM documents WHERE id = 'doc_B'
    ├─ [MySQL] UPDATE knowledgebases SET doc_num = doc_num - 1, ...
    └─ 返回 True
    ↓
GraphRAG 任务调度器检测到文档删除
    ↓
_save_global_graph() 清理旧数据
    ├─ [DocStore] DELETE knowledge_graph_kwd IN ['graph', 'subgraph'] WHERE kb_id = 'kb_001'
    ├─ [DocStore] DELETE knowledge_graph_kwd = 'entity' WHERE entity_kwd IN doc_B_entities
    └─ [DocStore] DELETE knowledge_graph_kwd = 'relation' WHERE (from, to) IN doc_B_edges
    ↓
clear_phase_markers(kb_001)
    ├─ [Redis] DEL graphrag:phase:kb_001:resolution_done
    └─ [Redis] DEL graphrag:phase:kb_001:community_done
    ↓
cleanup_checkpoints(tenant_001, kb_001, RESOLUTION_CHECKPOINT)
    ├─ [Redis] DEL graphrag:checkpoint:tenant_001:kb_001:graphrag_checkpoint_resolution:*
    └─ [Redis] DEL graphrag:checkpoint_index:tenant_001:kb_001:graphrag_checkpoint_resolution
    ↓
cleanup_checkpoints(tenant_001, kb_001, COMMUNITY_CHECKPOINT)
    ├─ [Redis] DEL graphrag:checkpoint:tenant_001:kb_001:graphrag_checkpoint_community:*
    └─ [Redis] DEL graphrag:checkpoint_index:tenant_001:kb_001:graphrag_checkpoint_community
    ↓
下次 GraphRAG 任务运行时
    ↓
run_graphrag_for_kb(tenant_001, kb_001)
    ├─ 检测 phase markers 缺失 → need_resolution = True, need_community = True
    ├─ 加载剩余子图（仅 doc_A）
    ├─ 合并子图 → global_graph
    ├─ 实体消解 → resolved_graph
    ├─ 社区检测 → communities
    ├─ 保存新的全局图 → DocStore
    ├─ 设置 phase markers → Redis
    └─ 返回成功
```

---

### 数据一致性保证

1. **原子性删除**：
   - 使用数据库事务 `DB.atomic()` 确保文档删除和 KB 计数器更新同时成功或同时失败
   - 使用行锁 `.for_update()` 防止并发删除

2. **幂等性设计**：
   - `delete_document_and_update_kb_counts()` 返回 False 如果文档已被删除
   - `clear_phase_markers()` 对不存在的 key 执行 DELETE 是安全的（Redis 空操作）
   - `cleanup_checkpoints()` 遍历实际存在的 keys，不存在时不会报错

3. **级联删除顺序**：
   - 先删除数据库记录（documents 表）
   - 再删除 DocStore 数据（子图、实体、关系）
   - 最后清理缓存（phase markers、checkpoints）
   - 这样即使中间步骤失败，下次任务会重新检测并清理

4. **重试机制**：
   - 边删除使用 3 次指数退避重试
   - `@retry_deadlock_operation()` 装饰器处理数据库死锁

---

## 场景 2：知识库清空

### 场景描述

**操作**：清空 KB `kb_001` 的所有 GraphRAG 数据，但保留 KB 结构（不删除 KB 本身）

**适用场景**：
- 用户想要重新构建知识图谱，但不想删除 KB 的配置和权限设置
- 批量删除所有文档后，清理残留的图数据

---

### 清空流程

**步骤 1：删除所有文档**
```python
from api.db.services.document_service import DocumentService

# 获取 KB 下的所有文档 ID
doc_ids = DocumentService.query(kb_id="kb_001").get_all_ids()

# 逐个删除文档
for doc_id in doc_ids:
    DocumentService.delete_document_and_update_kb_counts(doc_id)
```

**步骤 2：清空 DocStore 中的所有 GraphRAG 数据**
```python
# 删除所有 knowledge_graph_kwd 数据
await thread_pool_exec(
    settings.docStoreConn.delete,
    {"kb_id": "kb_001"},  # 删除所有属于该 KB 的数据
    search.index_name(tenant_id)
)
```

**步骤 3：清除所有 Redis 缓存**
```python
# 清除 phase markers
clear_phase_markers("kb_001")

# 清理 checkpoints
await cleanup_checkpoints(tenant_id, "kb_001", RESOLUTION_CHECKPOINT)
await cleanup_checkpoints(tenant_id, "kb_001", COMMUNITY_CHECKPOINT)

# 清除分布式锁（如果存在）
lock_key = f"graphrag_task_kb_001"
REDIS_CONN.delete(lock_key)
```

**数据变更**：
| 存储系统 | 数据项 | 变更 |
|----------|--------|------|
| MySQL/PostgreSQL | documents 表 | 所有 kb_id = 'kb_001' 的行被删除 |
| MySQL/PostgreSQL | knowledgebases 表 | doc_num = 0, chunk_num = 0, token_num = 0 |
| Elasticsearch/Infinity | knowledge_graph_kwd | 所有 kb_id = 'kb_001' 的数据被删除 |
| Redis | phase markers | 被清除 |
| Redis | checkpoints | 被清除 |
| Redis | 分布式锁 | 被释放 |

**KB 保留项**：
- KB 配置（名称、描述、权限、解析器配置等）
- KB ID 和租户关联关系

---

## 场景 3：知识库删除

### 场景描述

**操作**：完全删除 KB `kb_001` 及其所有关联数据

**适用场景**：
- 用户不再需要该 KB
- 清理测试数据

---

### 删除流程

**步骤 1：删除所有文档**（同场景 2）

**步骤 2：删除 DocStore 数据**（同场景 2）

**步骤 3：清除 Redis 缓存**（同场景 2）

**步骤 4：删除 KB 本身**
```python
from api.db.services.knowledgebase_service import KnowledgebaseService

kb_id = "kb_001"
KnowledgebaseService.delete_by_id(kb_id)
```

**数据变更**：
| 存储系统 | 数据项 | 变更 |
|----------|--------|------|
| MySQL/PostgreSQL | knowledgebases 表 | kb_id = 'kb_001' 的行被删除 |
| 其他 | 同场景 2 | 同场景 2 |

**完全删除清单**：
- ✅ KB 配置行
- ✅ 所有文档行
- ✅ 所有 DocStore 图数据
- ✅ 所有 Redis 缓存
- ✅ 所有权限关联（如果有单独的权限表）

---

## 场景 4：仅清理缓存

### 场景描述

**操作**：清理 Redis 缓存以强制重新计算，但保留 DocStore 中的数据

**适用场景**：
- 调试或测试时，想要重新运行实体消解或社区检测
- 怀疑缓存数据不一致，想要强制重建
- 修改了抽取器配置（如切换 LLM 模型），想要重新生成图谱

---

### 清理流程

**步骤 1：清除 Phase Markers**
```python
from rag.graphrag.phase_markers import clear_phase_markers

kb_id = "kb_001"
clear_phase_markers(kb_id)  # 清除 resolution_done 和 community_done
```

**步骤 2：清理 Checkpoints**
```python
from rag.graphrag.checkpoints import cleanup_checkpoints, COMMUNITY_CHECKPOINT, RESOLUTION_CHECKPOINT

tenant_id = "tenant_001"
kb_id = "kb_001"

await cleanup_checkpoints(tenant_id, kb_id, RESOLUTION_CHECKPOINT)
await cleanup_checkpoints(tenant_id, kb_id, COMMUNITY_CHECKPOINT)
```

**步骤 3：（可选）释放分布式锁**
```python
from rag.utils.redis_conn import REDIS_CONN

lock_key = f"graphrag_task_{kb_id}"
REDIS_CONN.delete(lock_key)
```

**数据保留**：
- ✅ documents 表：所有文档行保留
- ✅ knowledgebases 表：KB 行保留
- ✅ DocStore：所有 subgraph、entity、relation、community_report 数据保留

**数据清除**：
- ❌ Redis phase markers：被清除
- ❌ Redis checkpoints：被清除
- ❌ Redis 分布式锁：被释放

**下次任务行为**：
- 检测到 phase markers 缺失
- 从 DocStore 加载现有的 subgraph 数据
- 重新执行实体消解和社区检测
- 覆盖旧的 entity、relation、community_report 数据

---

### 使用脚本示例

```python
# scripts/clear_graphrag_cache.py

import asyncio
from rag.graphrag.phase_markers import clear_phase_markers
from rag.graphrag.checkpoints import cleanup_checkpoints, RESOLUTION_CHECKPOINT, COMMUNITY_CHECKPOINT
from rag.utils.redis_conn import REDIS_CONN

async def clear_cache_for_kb(tenant_id: str, kb_id: str):
    """清理指定 KB 的 GraphRAG 缓存"""
    
    print(f"Clearing GraphRAG cache for KB: {kb_id}")
    
    # 1. 清除 phase markers
    print("  - Clearing phase markers...")
    clear_phase_markers(kb_id)
    
    # 2. 清理 checkpoints
    print("  - Cleaning up resolution checkpoints...")
    success_resolution = await cleanup_checkpoints(tenant_id, kb_id, RESOLUTION_CHECKPOINT)
    
    print("  - Cleaning up community checkpoints...")
    success_community = await cleanup_checkpoints(tenant_id, kb_id, COMMUNITY_CHECKPOINT)
    
    # 3. 释放分布式锁
    print("  - Releasing distributed lock...")
    lock_key = f"graphrag_task_{kb_id}"
    REDIS_CONN.delete(lock_key)
    
    # 4. 汇总结果
    if success_resolution and success_community:
        print(f"✅ Successfully cleared cache for KB: {kb_id}")
        print("   Next GraphRAG run will re-execute entity resolution and community detection.")
    else:
        print(f"⚠️  Partially cleared cache for KB: {kb_id}")
        print(f"   Resolution cleanup: {'✅' if success_resolution else '❌'}")
        print(f"   Community cleanup: {'✅' if success_community else '❌'}")

if __name__ == "__main__":
    tenant_id = "tenant_001"
    kb_id = "kb_001"
    
    asyncio.run(clear_cache_for_kb(tenant_id, kb_id))
```

**运行示例**：
```bash
$ python scripts/clear_graphrag_cache.py

Clearing GraphRAG cache for KB: kb_001
  - Clearing phase markers...
  - Cleaning up resolution checkpoints...
  - Cleaning up community checkpoints...
  - Releasing distributed lock...
✅ Successfully cleared cache for KB: kb_001
   Next GraphRAG run will re-execute entity resolution and community detection.
```

---

## 性能考量

### 单文档删除

| 操作 | 耗时 | 瓶颈 |
|------|------|------|
| 文档行删除 + KB 计数器更新 | < 100ms | 数据库事务 |
| 子图数据删除（DocStore） | 500ms - 2s | DocStore 删除操作 |
| Phase markers 清除 | < 50ms | Redis 操作 |
| Checkpoints 清理 | 100ms - 1s | Redis 遍历和删除 |
| 全局图重建 | 30s - 5min | LLM 调用（实体消解、社区检测） |

**优化建议**：
- 子图删除使用批量操作（100 个节点/批次）
- 边删除并发执行但受限流器保护
- Checkpoints 清理使用分页遍历，避免一次性加载所有 keys

### 知识库清空

| 操作 | 耗时 | 瓶颈 |
|------|------|------|
| 删除所有文档 | N × 100ms | 顺序删除，N = 文档数量 |
| 清空 DocStore | 2s - 10s | DocStore 批量删除 |
| 清除 Redis 缓存 | < 1s | Redis 操作 |

**优化建议**：
- 可以并发删除文档，但需注意数据库连接池限制
- DocStore 清空可以使用 `delete_by_query` 一次性删除所有数据

### 仅清理缓存

| 操作 | 耗时 | 瓶颈 |
|------|------|------|
| 清除 phase markers | < 50ms | Redis 操作 |
| 清理 checkpoints | 100ms - 1s | Redis 遍历 |
| 释放分布式锁 | < 10ms | Redis 操作 |

**性能影响**：几乎无影响，适合频繁使用

---

## 错误处理

### 并发删除保护

**问题**：多个请求同时删除同一文档，导致 KB 计数器重复扣减

**解决方案**：
```python
# document_service.py 使用行锁和幂等性设计
doc = cls.model.select(...).where(...).for_update().get_or_none()
if doc is None:
    return False  # 已被删除，直接返回
```

### Phase Markers 清除失败

**问题**：Redis 连接失败，phase markers 未被清除

**影响**：下次任务不会重新执行实体消解和社区检测，图谱不会更新

**解决方案**：
```python
# phase_markers.py 捕获异常并记录日志
try:
    REDIS_CONN.delete(_phase_key(kb_id, phase))
except Exception:
    logging.exception("clear_phase_markers(%s, %s) failed", kb_id, phase)
```

**恢复方法**：手动调用 `clear_phase_markers(kb_id)` 重试

### Checkpoints 清理失败

**问题**：部分 checkpoints 未被删除

**影响**：下次任务可能使用旧的 checkpoints，导致数据不一致

**解决方案**：
```python
# checkpoints.py 返回布尔值表示成功/失败
try:
    # ... 清理逻辑 ...
    return True
except Exception:
    logging.exception("Failed to cleanup GraphRAG checkpoints type=%s kb=%s", checkpoint_type, kb_id)
    return False
```

**恢复方法**：
- 检查日志确认失败原因
- 手动调用 `cleanup_checkpoints()` 重试
- 或使用 Redis CLI 手动删除：`DEL graphrag:checkpoint:*:kb_001:*`

### DocStore 删除失败

**问题**：DocStore 连接失败或删除超时

**影响**：旧的子图数据残留，可能导致全局图包含已删除文档的实体

**解决方案**：
```python
# utils.py 边删除使用重试机制
max_retries = 3
for attempt in range(max_retries):
    try:
        await thread_pool_exec(settings.docStoreConn.delete, ...)
        return
    except Exception as e:
        if attempt < max_retries - 1:
            wait = 2**attempt
            await asyncio.sleep(wait)
        else:
            raise
```

**恢复方法**：
- 检查 DocStore（Elasticsearch/Infinity）健康状态
- 手动清理残留数据（使用 DocStore 管理工具）
- 重新触发全局图重建

---

## 总结

### 四种场景对比

| 场景 | 文档数据 | DocStore 图数据 | Redis 缓存 | KB 配置 |
|------|----------|----------------|------------|---------|
| 单文档删除 | 删除 1 个文档 | 删除该文档子图 | 清除 | 保留 |
| 知识库清空 | 删除所有文档 | 清空所有图数据 | 清除 | 保留 |
| 知识库删除 | 删除所有文档 | 清空所有图数据 | 清除 | 删除 |
| 仅清理缓存 | 保留 | 保留 | 清除 | 保留 |

### 关键函数索引

| 函数 | 文件路径 | 行号 | 功能 |
|------|----------|------|------|
| `delete_document_and_update_kb_counts` | `api/db/services/document_service.py` | 847-875 | 原子性删除文档并更新 KB 计数器 |
| `clear_phase_markers` | `rag/graphrag/phase_markers.py` | 77-85 | 清除 phase markers |
| `cleanup_checkpoints` | `rag/graphrag/checkpoints.py` | 120-134 | 清理 checkpoints |
| `_save_global_graph` | `rag/graphrag/utils.py` | 680-779 | 删除旧图数据并保存新图 |
| `run_graphrag_for_kb` | `rag/graphrag/general/index.py` | 150-300 | GraphRAG 主入口，检测 phase markers 并触发重建 |

### 最佳实践

1. **删除前备份**：重要 KB 删除前先导出数据
2. **分阶段清理**：先清理缓存，观察重建效果，再决定是否删除数据
3. **监控日志**：关注删除和重建过程的日志，及时发现异常
4. **性能测试**：大规模删除前先在测试环境验证性能
5. **幂等性利用**：删除操作支持重复调用，失败时可以安全重试
6. **级联清理**：确保 phase markers 和 checkpoints 都被清理，避免数据不一致
7. **锁的释放**：删除或清理后手动释放分布式锁，避免阻塞下次任务

---

**文档版本**：1.0  
**最后更新**：2026-09-18  
**适用代码版本**：RAGFlow 主分支（2026-09）
