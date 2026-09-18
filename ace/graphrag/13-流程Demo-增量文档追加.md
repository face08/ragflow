# GraphRAG 流程 Demo：增量文档追加

## 场景描述

**知识库状态**：`kb_001` 已有一篇文档 `doc_A.txt` 的 GraphRAG 数据
- 现有节点数：44 个实体
- 现有边数：78 条关系
- 现有社区数：5 个社区

**新增文档**：`doc_B.txt`（2500 tokens，关于"机器学习模型训练"）
- 文档 ID：`doc_B_id_456`
- 预期：部分实体与 doc_A 重叠（如"神经网络"、"数据集"等）

**配置参数**：
```python
kb_id = "kb_001"
tenant_id = "tenant_xyz"
language = "Chinese"
chat_model = "gpt-4o"
embedding_model = "text-embedding-3-large"
```

---

## 阶段 0：准备 - 加载与批处理分块

### 处理逻辑

1. 从 DocStore 加载新文档的所有分块
2. 按 token 数量批处理合并（每批 ≤4096 tokens）

### Demo 数据示例

#### 0.1 加载分块

```python
from rag.graphrag.general.index import get_chunks

chunks = await get_chunks(tenant_id="tenant_xyz", doc_ids=["doc_B_id_456"])
# 返回 10 个分块
```

**示例 chunks[0]**：
```json
{
  "id": "chunk_doc_B_456_0",
  "content_with_weight": "机器学习模型训练需要大量数据集。神经网络通过反向传播算法优化参数...",
  "content_ltks": "机器 学习 模型 训练 需要 大量 数据集 神经 网络...",
  "kb_id": ["kb_001"],
  "doc_id": "doc_B_id_456",
  "docnm_kwd": "doc_B.txt",
  "token_num": 245
}
```

**所有分块统计**：
```python
total_chunks = 10
total_tokens = sum(chunk["token_num"] for chunk in chunks)  # 2500 tokens
```

#### 0.2 批处理合并

```python
from rag.graphrag.general.index import _batch_chunks

batched_chunks = _batch_chunks(chunks, max_tokens=4096)
# 结果：3 个批次
```

**批次分布**：
- `batch_0`: chunks[0:4]（token_num: 245+260+250+245 = 1000 tokens）
- `batch_1`: chunks[4:8]（token_num: 240+255+250+255 = 1000 tokens）
- `batch_2`: chunks[8:10]（token_num: 250+250 = 500 tokens）

### 输出
- **内存对象**：`batched_chunks: List[List[dict]]`（3 个批次）
- **无外部存储**

---

## 阶段 1：子图生成 - 实体关系抽取

### 处理逻辑

1. 并发处理 3 个批次（`max_parallel_docs=4`）
2. 每批次调用 LLM 抽取实体和关系
3. 合并所有子图为文档级子图

### Demo 数据示例

#### 1.1 并发抽取子图

```python
from rag.graphrag.light.graph_extractor import LightKGExt
import asyncio

extractor = LightKGExt(
    language="Chinese",
    tuple_delimiter="<|>",
    record_delimiter="##",
    completion_delimiter="<|COMPLETE|>"
)

subgraphs = await asyncio.gather(
    *[extractor.extract(batch, chat_model="gpt-4o") for batch in batched_chunks]
)

# 返回 3 个 nx.Graph
# subgraph_0: 15 nodes, 22 edges
# subgraph_1: 18 nodes, 28 edges
# subgraph_2: 5 nodes, 8 edges
```

#### 1.2 LLM 调用示例（batch_0）

**Prompt**：
```
-Goal-
给定一段可能与此活动相关的文本，识别所有实体和关系。
输出格式使用 ("<|>", "##", "<|COMPLETE|>") 作为分隔符。

-Examples-
("entity"<|>机器学习<|>CONCEPT<|>一种让计算机从数据中学习的技术)##
("relationship"<|>机器学习<|>神经网络<|>机器学习是神经网络的上层概念<|>包含,使用<|>8)##

-Real Data-
机器学习模型训练需要大量数据集。神经网络通过反向传播算法优化参数...
（batch_0 的 4 个分块合并文本，共 1000 tokens）
```

**LLM Response**（delimiter 格式）：
```
("entity"<|>机器学习<|>CONCEPT<|>一种让计算机从数据中学习的技术，广泛应用于预测、分类等任务)##
("entity"<|>神经网络<|>ALGORITHM<|>模拟人脑神经元结构的机器学习算法，由多层神经元组成)##
("entity"<|>数据集<|>RESOURCE<|>用于训练和测试机器学习模型的数据集合)##
("entity"<|>反向传播<|>ALGORITHM<|>神经网络训练中用于计算梯度和更新参数的核心算法)##
("entity"<|>参数优化<|>PROCESS<|>通过调整模型参数来最小化损失函数的过程)##
("relationship"<|>机器学习<|>神经网络<|>机器学习是神经网络的上层概念<|>包含,使用<|>8)##
("relationship"<|>神经网络<|>数据集<|>神经网络训练需要大量数据集<|>需要,依赖<|>9)##
("relationship"<|>神经网络<|>反向传播<|>神经网络通过反向传播算法优化<|>使用,依赖<|>9)##
("relationship"<|>反向传播<|>参数优化<|>反向传播是实现参数优化的关键方法<|>实现,支持<|>8)##
<|COMPLETE|>
```

#### 1.3 解析为 NetworkX 图

```python
# graph_extractor.py 内部逻辑
import networkx as nx

subgraph_0 = nx.Graph()

# 添加节点（实体名统一 .upper() 标准化）
subgraph_0.add_node("机器学习", entity_type="CONCEPT", description="一种让计算机从数据中学习的技术，广泛应用于预测、分类等任务", source_id=["doc_B_id_456"])
subgraph_0.add_node("神经网络", entity_type="ALGORITHM", description="模拟人脑神经元结构的机器学习算法，由多层神经元组成", source_id=["doc_B_id_456"])
subgraph_0.add_node("数据集", entity_type="RESOURCE", description="用于训练和测试机器学习模型的数据集合", source_id=["doc_B_id_456"])
# ... 其他 12 个节点

# 添加边（边键自动排序：sorted([src, tgt])）
subgraph_0.add_edge("机器学习", "神经网络", 
                    description="机器学习是神经网络的上层概念",
                    keywords="包含,使用",
                    weight=8.0,
                    source_id=["doc_B_id_456"])
subgraph_0.add_edge("数据集", "神经网络",
                    description="神经网络训练需要大量数据集",
                    keywords="需要,依赖",
                    weight=9.0,
                    source_id=["doc_B_id_456"])
# ... 其他 20 条边

# 结果：subgraph_0 有 15 nodes, 22 edges
```

#### 1.4 合并子图

```python
import networkx as nx

doc_B_subgraph = nx.compose_all([subgraph_0, subgraph_1, subgraph_2])
# 结果：38 nodes, 58 edges
```

**重叠实体处理**（compose_all 自动合并）：
- "神经网络" 在 batch_0 和 batch_1 都出现 → description 自动拼接（带 `<SEP>` 分隔符）
- source_id 保持为 `["doc_B_id_456"]`（同一文档，不重复）

### 输出

### 输出

**DocStore（Elasticsearch）**：
```json
// knowledge_graph_kwd="subgraph", source_id="doc_B_id_456"
{
  "id": "chunk_subgraph_doc_B_id_456_abc123",
  "kb_id": ["kb_001"],
  "knowledge_graph_kwd": "subgraph",
  "source_id": "doc_B_id_456",
  "content_with_weight": "机器学习^8 神经网络^9 数据集^9 反向传播^9 参数优化^8 ...",
  "graph_json": "{\"nodes\": [{\"id\": \"机器学习\", \"entity_type\": \"CONCEPT\", ...}], \"edges\": [{\"source\": \"机器学习\", \"target\": \"神经网络\", ...}]}"
}
```

**Redis 缓存**：
```bash
# LLM 缓存（24h TTL）
graphrag_llm_cache:a3f8e9d2... → {"entities": [...], "relationships": [...]}

# Embed 缓存（24h TTL）
graphrag_embed_cache:机器学习 → [0.123, -0.456, 0.789, ...]
graphrag_embed_cache:神经网络 → [-0.234, 0.567, -0.891, ...]
# ... 其他 36 个实体嵌入
```

**关键差异点 vs 首次建图**：
- ✅ 无差异：子图生成过程完全相同
- 新文档的子图独立生成，不依赖现有图谱

---

## 阶段 2：子图合并 - 增量更新全局知识图谱

### 处理逻辑

1. 从 DocStore 加载现有全局图
2. 使用 `graph_merge` 合并 doc_B 子图到全局图
3. 删除旧的 graph/subgraph chunks
4. 插入新的全局图和子图 chunks

### Demo 数据示例

#### 2.1 加载现有全局图

```python
from rag.graphrag.general.index import get_graph

existing_graph = await get_graph(tenant_id="tenant_xyz", kb_id="kb_001")
print(f"现有图: {existing_graph.number_of_nodes()} 节点, {existing_graph.number_of_edges()} 边")
# 输出: 现有图: 44 节点, 78 边
```

**现有图部分节点示例**：
```python
# 节点 "神经网络" 已存在（来自 doc_A）
existing_graph.nodes["神经网络"] = {
    "entity_type": "ALGORITHM",
    "description": "一种模拟生物神经网络的计算模型<SEP>用于图像识别和自然语言处理",
    "source_id": ["doc_A_id_123"]
}

# 节点 "数据集" 已存在
existing_graph.nodes["数据集"] = {
    "entity_type": "RESOURCE", 
    "description": "用于训练AI模型的标注数据集合",
    "source_id": ["doc_A_id_123"]
}
```

#### 2.2 执行图合并

```python
from rag.graphrag.general.index import graph_merge
from rag.graphrag.utils import GraphChange

merged_graph, change = graph_merge(existing_graph, doc_B_subgraph)
print(f"合并后: {merged_graph.number_of_nodes()} 节点, {merged_graph.number_of_edges()} 边")
# 输出: 合并后: 77 节点, 128 边
```

**计算说明**：
- 节点: 44（原有）+ 38（doc_B）- 5（重叠）= 77
- 边: 78（原有）+ 58（doc_B）- 8（重叠）= 128
- 重叠实体: "神经网络", "数据集", "机器学习", "算法", "模型"（5 个）

**重叠实体合并示例**（"神经网络"）：
```python
# 合并前
doc_A: "一种模拟生物神经网络的计算模型<SEP>用于图像识别和自然语言处理"
doc_B: "模拟人脑神经元结构的机器学习算法，由多层神经元组成"

# 合并后（使用 GRAPH_FIELD_SEP = "<SEP>"）
merged_graph.nodes["神经网络"]["description"] = (
    "一种模拟生物神经网络的计算模型<SEP>用于图像识别和自然语言处理<SEP>"
    "模拟人脑神经元结构的机器学习算法，由多层神经元组成"
)
merged_graph.nodes["神经网络"]["source_id"] = ["doc_A_id_123", "doc_B_id_456"]
```

**重叠边合并示例**（"神经网络" - "数据集"）：
```python
# 合并前
doc_A 边: description="神经网络需要大规模数据集进行训练", weight=7.5
doc_B 边: description="神经网络训练需要大量数据集", weight=9.0

# 合并后（description 拼接，weight 取 max）
merged_graph["神经网络"]["数据集"]["description"] = (
    "神经网络需要大规模数据集进行训练<SEP>神经网络训练需要大量数据集"
)
merged_graph["神经网络"]["数据集"]["weight"] = 9.0  # max(7.5, 9.0)
```

#### 2.3 记录变更

```python
# GraphChange 对象
change = GraphChange(
    removed_nodes=set(),  # 增量追加不删除节点
    added_updated_nodes={"机器学习", "神经网络", "数据集", ..., "优化器"},  # 38 个
    removed_edges=set(),
    added_updated_edges={("机器学习", "神经网络"), ("神经网络", "数据集"), ...}  # 58 条
)
```

#### 2.4 更新存储

```python
from rag.graphrag.utils import set_graph

# 调用 set_graph 执行 4 项存储操作
await set_graph(
    tenant_id="tenant_xyz",
    kb_id="kb_001",
    graph=merged_graph,
    change=change,
    doc_store=ELASTICSEARCH,
    embedding_model=embedding_model,
    redis_client=redis
)
```

**存储操作说明**：

1. **删除旧的 graph/subgraph chunks**：
   ```python
   await delete_by_query(
       tenant_id="tenant_xyz",
       kb_id="kb_001",
       knowledge_graph_kwd=["graph", "subgraph"]
   )
   ```

2. **插入新的全局图 chunks**（77 节点 + 128 边 = 205 chunks）：
   ```json
   // 节点 chunk 示例（"神经网络"，合并后）
   {
     "id": "chunk_graph_kb_001_node_神经网络",
     "kb_id": ["kb_001"],
     "knowledge_graph_kwd": "graph",
     "name": "神经网络",
     "name_kwd": "神经网络",
     "content_with_weight": "神经网络 ALGORITHM",
     "source_id": ["doc_A_id_123", "doc_B_id_456"],
     "description": "一种模拟生物神经网络的计算模型<SEP>用于图像识别和自然语言处理<SEP>模拟人脑神经元结构的机器学习算法，由多层神经元组成"
   }
   
   // 边 chunk 示例（"神经网络" - "数据集"，合并后）
   {
     "id": "chunk_graph_kb_001_edge_数据集_神经网络",
     "kb_id": ["kb_001"],
     "knowledge_graph_kwd": "graph",
     "content_with_weight": "神经网络 需要 数据集",
     "source_id": ["doc_A_id_123", "doc_B_id_456"],
     "src_id": "神经网络",
     "tgt_id": "数据集",
     "description": "神经网络需要大规模数据集进行训练<SEP>神经网络训练需要大量数据集",
     "weight": 9.0
   }
   ```

3. **插入新文档的子图 chunk**（doc_B）：
   ```json
   {
     "id": "chunk_subgraph_doc_B_id_456_abc123",
     "kb_id": ["kb_001"],
     "knowledge_graph_kwd": "subgraph",
     "source_id": "doc_B_id_456",
     "graph_json": "{\"nodes\": [{\"id\": \"机器学习\", ...}], \"edges\": [...]}"
   }
   ```

4. **批量预热嵌入缓存**：
   - 对 38 个新增/更新实体调用 embedding 模型
   - 其中 5 个重叠实体命中 Redis 缓存，实际只需计算 33 个新嵌入

### 输出

**DocStore 状态**：
- 全局图 chunks: 205（77 nodes + 128 edges）
- 子图 chunks: 2（doc_A + doc_B）

**Redis Phase Marker**：
```bash
graphrag_phase:kb_001 → "PHASE_MERGE_DONE"
TTL: 7 days
```

**关键差异点 vs 首次建图**：
- ✅ **重叠处理**：新旧实体的 description 用 `<SEP>` 拼接，source_id 数组合并
- ✅ **增量更新**：在现有图基础上 merge，而非全新创建
- ✅ **嵌入缓存复用**：重叠实体命中缓存，节省 embedding 成本
- ⚠️ **存储策略相同**：都是先删除旧 graph/subgraph，再插入新数据

---

## 阶段 3：实体消解 - 合并相似实体

### 处理逻辑

1. 使用 rapidfuzz Levenshtein 预过滤相似实体对（阈值 ≥ 0.85）
2. 批量提交给 LLM 判断是否为同一实体（每批 10 对）
3. 合并确认的实体对，更新全局图
4. 保存 checkpoint 到 Redis

### Demo 数据示例

#### 3.1 预过滤相似实体

```python
from rapidfuzz import fuzz
import itertools

# 计算所有实体对的 Levenshtein 相似度
similar_pairs = []
for e1, e2 in itertools.combinations(merged_graph.nodes(), 2):
    similarity = fuzz.ratio(e1, e2) / 100.0
    if similarity >= 0.85:
        similar_pairs.append((e1, e2, similarity))

# 结果：找到 12 对候选
print(f"候选实体对: {len(similar_pairs)}")
```

**候选对示例**：
```python
[
    ("神经网络", "神经网络模型", 0.92),
    ("数据集", "训练数据", 0.87),
    ("机器学习", "机器学习算法", 0.95),
    ("反向传播", "反向传播算法", 0.93),
    ("优化器", "优化算法", 0.88),
    ("参数优化", "参数调整", 0.91),
    # ... 其他 6 对
]
```

#### 3.2 LLM 判断批次（批次 1）

**Prompt**（前 10 对）：
```
以下是可能指代同一实体的候选对，请判断每对是否应该合并。

Pair 0:
- Entity 1: "神经网络"
  Description: 一种模拟生物神经网络的计算模型<SEP>用于图像识别和自然语言处理<SEP>模拟人脑神经元结构的机器学习算法，由多层神经元组成
- Entity 2: "神经网络模型"
  Description: 指具体的神经网络架构实例，如 CNN、RNN 等

Pair 1:
- Entity 1: "数据集"
  Description: 用于训练AI模型的标注数据集合<SEP>用于训练和测试机器学习模型的数据集合
- Entity 2: "训练数据"
  Description: 用于训练机器学习模型的数据样本

...

请输出 JSON 格式：
{
**LLM Response**：
```json
{
  "resolutions": [
    {"pair": 0, "same": false, "reason": "虽然名称相似，但'神经网络'是通用概念，'神经网络模型'指具体架构实例（如CNN、RNN），应保持独立"},
    {"pair": 1, "same": true, "reason": "'数据集'和'训练数据'都指用于训练的数据集合，语义完全一致，应合并为'数据集'"},
    {"pair": 2, "same": true, "reason": "'机器学习'和'机器学习算法'本质相同，后者只是前者的完整表述，应合并为'机器学习'"},
    {"pair": 3, "same": false, "reason": "'反向传播'是算法名称，'反向传播算法'强调算法属性，但实际指代相同事物，应合并为'反向传播'"},
    {"pair": 4, "same": true, "reason": "'优化器'和'优化算法'在机器学习语境中通常可互换，应合并为'优化器'"},
    {"pair": 5, "same": true, "reason": "'参数优化'和'参数调整'都指调整模型参数的过程，应合并为'参数优化'"},
    {"pair": 6, "same": true, "reason": "..."},
    {"pair": 7, "same": false, "reason": "..."},
    {"pair": 8, "same": true, "reason": "..."},
    {"pair": 9, "same": true, "reason": "..."}
  ]
}
```

**批次 1 结果**：10 对中确认合并 7 对

#### 3.3 执行图合并

```python
# 对确认的实体对执行合并
confirmed_pairs = [
    ("数据集", "训练数据"),
    ("机器学习", "机器学习算法"),
    ("反向传播", "反向传播算法"),
    ("优化器", "优化算法"),
    ("参数优化", "参数调整"),
    # ... 另外 3 对
]

for canonical_name, alias_name in confirmed_pairs:
    # 合并节点：保留 canonical_name，删除 alias_name
    canonical_data = merged_graph.nodes[canonical_name]
    alias_data = merged_graph.nodes[alias_name]
    
    # 合并 description
    canonical_data["description"] += GRAPH_FIELD_SEP + alias_data["description"]
    
    # 合并 source_id
    canonical_data["source_id"] = list(set(canonical_data["source_id"] + alias_data["source_id"]))
    
    # 重定向所有指向 alias_name 的边
    for neighbor in list(merged_graph.neighbors(alias_name)):
        if merged_graph.has_edge(canonical_name, neighbor):
            # 边已存在，合并 description 和 weight
            edge_old = merged_graph[canonical_name][neighbor]
            edge_new = merged_graph[alias_name][neighbor]
            edge_old["description"] += GRAPH_FIELD_SEP + edge_new["description"]
            edge_old["weight"] = max(edge_old["weight"], edge_new["weight"])
        else:
            # 新边，直接添加
            merged_graph.add_edge(canonical_name, neighbor, **merged_graph[alias_name][neighbor])
    
    # 删除别名节点
    merged_graph.remove_node(alias_name)

print(f"消解后: {merged_graph.number_of_nodes()} 节点, {merged_graph.number_of_edges()} 边")
# 输出: 消解后: 69 节点, 120 边
```

**计算说明**：
- 节点: 77 - 8（合并的别名）= 69
- 边: 128 - 8（重复边合并）= 120

#### 3.4 保存检查点

```python
import hashlib
import json

checkpoint_key = "graphrag_checkpoint:entity:kb_001"

for canonical_name, alias_name in confirmed_pairs:
    pair_key = canonical_name + GRAPH_FIELD_SEP + alias_name
    field = hashlib.sha256(pair_key.encode()).hexdigest()
    
    await redis.hset(checkpoint_key, field, json.dumps({
        "same": True,
        "kept": canonical_name,
        "removed": alias_name,
        "timestamp": "2026-09-18 14:30:00"
    }))

await redis.expire(checkpoint_key, 7 * 86400)

# Phase marker
await redis.setex("graphrag_phase:kb_001", 7*24*3600, "PHASE_RESOLUTION")
```

#### 3.5 更新存储

```python
from rag.graphrag.utils import set_graph

await set_graph(
    tenant_id="tenant_xyz",
    kb_id="kb_001",
    graph=merged_graph,  # 69 节点, 120 边
    change=GraphChange(
        removed_nodes={"训练数据", "机器学习算法", "反向传播算法", ...},  # 8 个别名
        added_updated_nodes={"数据集", "机器学习", "反向传播", ...},  # 8 个保留实体
        removed_edges={...},
        added_updated_edges={...}
    ),
    doc_store=ELASTICSEARCH,
    embedding_model=embedding_model,
    redis_client=redis
)
```

### 输出

**DocStore 状态**：
- 全局图 chunks: 189（69 nodes + 120 edges）
- entity chunks（带别名）:
  ```json
  {
    "id": "chunk_entity_kb_001_数据集",
    "kb_id": ["kb_001"],
    "knowledge_graph_kwd": "entity",
    "name": "数据集",
    "name_kwd": "数据集",
    "description": "用于训练AI模型的标注数据集合<SEP>用于训练和测试机器学习模型的数据集合<SEP>用于训练机器学习模型的数据样本",
    "source_id": ["doc_A_id_123", "doc_B_id_456"],
    "degree": 15,
    "aliases": ["训练数据"]
  }
  ```

**Redis Checkpoint**：
```bash
graphrag_checkpoint:entity:kb_001 → Hash with 8 fields
# field: sha256("数据集<SEP>训练数据")
# value: {"same": true, "kept": "数据集", "removed": "训练数据", "timestamp": "..."}

graphrag_phase:kb_001 → "PHASE_RESOLUTION"
TTL: 7 days
```

**关键差异点 vs 首次建图**：
- ✅ **更多候选对**：因为节点数增加（77 vs 44），相似对更多（12 vs 6）
- ✅ **跨文档合并**：可能合并来自 doc_A 和 doc_B 的不同实体
- ⚠️ **算法相同**：rapidfuzz 预过滤 + LLM 判断的流程不变

---

## 阶段 4：社区检测 - 重新运行 Leiden 算法

### 处理逻辑

1. 查询 Phase Marker，确认从哪个阶段开始
2. 对整个全局图运行 Leiden 社区检测
3. 为每个社区并发生成社区报告（LLM）
4. 构建社区报告 chunks 并批量插入

### Demo 数据示例

#### 4.1 运行 Leiden 社区检测

```python
from rag.graphrag.general.leiden import run as leiden_run

community_mapping = leiden_run(merged_graph, {})
# 返回: {node_name: community_id}
```

**社区划分结果**：
```python
# 检测到 6 个社区（原 5 个 + 新增 1 个）
from collections import defaultdict

communities = defaultdict(list)
for entity, comm_id in community_mapping.items():
    communities[comm_id].append(entity)

print(f"检测到 {len(communities)} 个社区")
# 输出: 检测到 6 个社区

# 社区分布
for comm_id, entities in communities.items():
    print(f"社区 {comm_id}: {len(entities)} 个实体")
```

**社区划分示例**：
```python
# 社区 0: 12 个实体（机器学习核心概念）
communities[0] = ["机器学习", "神经网络", "数据集", "反向传播", "参数优化", 
                  "梯度下降", "损失函数", "激活函数", "过拟合", "正则化", "验证集", "测试集"]

# 社区 1: 10 个实体（深度学习模型）
communities[1] = ["深度学习", "卷积神经网络", "循环神经网络", "Transformer", 
                  "ResNet", "BERT", "GPT", "注意力机制", "编码器", "解码器"]

# 社区 2: 8 个实体（优化与训练策略）← 新增社区
communities[2] = ["优化器", "Adam", "SGD", "学习率", "批次大小", "训练策略", "预训练", "微调"]

# 社区 3-5: 其他社区（略）
```

#### 4.2 为社区 5（新增社区）生成报告

**社区 5 子图构建**：
```python
community_5_entities = communities[5]
community_5_graph = merged_graph.subgraph(community_5_entities)
print(f"社区 5 子图: {community_5_graph.number_of_nodes()} 节点, {community_5_graph.number_of_edges()} 边")
# 输出: 社区 5 子图: 8 节点, 12 边
```

**LLM 生成社区报告**：

步骤 1：提取实体和关系信息
```python
entities_info = []
for entity in community_5_entities:
    node_data = community_5_graph.nodes[entity]
    entities_info.append({
        "name": entity,
        "type": node_data["entity_type"],
        "description": node_data["description"],
        "degree": community_5_graph.degree(entity)
    })

relationships_info = []
for src, tgt in community_5_graph.edges():
    edge_data = community_5_graph[src][tgt]
    relationships_info.append({
        "source": src,
        "target": tgt,
        "description": edge_data["description"],
        "weight": edge_data["weight"]
    })
```

步骤 2：构建 LLM Prompt
```
请为以下社区生成一份分析报告。

实体列表:
- 优化器 (COMPONENT): 机器学习中用于更新模型参数的算法组件
- Adam (ALGORITHM): 自适应学习率优化算法，结合动量和RMSProp
- SGD (ALGORITHM): 随机梯度下降，最基础的优化算法
- 学习率 (PARAMETER): 控制参数更新步长的超参数
- 批次大小 (PARAMETER): 每次训练迭代使用的样本数量
- 训练策略 (CONCEPT): 模型训练过程中的整体规划和方法
- 预训练 (PROCESS): 在大规模数据上预先训练模型的过程
- 微调 (PROCESS): 在特定任务上调整预训练模型的过程

关系列表:
- 优化器 → Adam (包含关系, weight=8.5)
- 优化器 → SGD (包含关系, weight=8.0)
- Adam → 学习率 (依赖关系, weight=9.0)
- SGD → 学习率 (依赖关系, weight=8.5)
- 训练策略 → 批次大小 (包含关系, weight=7.5)
- 训练策略 → 预训练 (包含关系, weight=8.0)
- 预训练 → 微调 (顺序关系, weight=9.0)
...

请输出 JSON 格式的报告，包含: title, summary, findings (每个 finding 含 summary/explanation/rating)
```

步骤 3：LLM 响应
```json
{
  "title": "模型训练优化策略社区",
  "summary": "该社区聚焦于机器学习模型训练的优化策略和超参数调整，涵盖优化器选择、学习率设置、批次大小配置以及预训练-微调范式等核心训练技术。",
  "findings": [
    {
      "summary": "优化器是训练效率的关键决定因素",
      "explanation": "Adam 和 SGD 是两种主流优化器。Adam 通过自适应学习率在多数场景表现优异，SGD 虽简单但在某些任务上泛化性能更好。",
      "rating": 8
    },
    {
      "summary": "学习率是最重要的超参数之一",
      "explanation": "所有优化器都依赖学习率控制更新步长。过大导致震荡，过小导致收敛缓慢。自适应优化器能部分缓解此问题。",
      "rating": 9
    },
    {
      "summary": "预训练-微调范式成为主流",
      "explanation": "大规模预训练捕获通用知识，下游任务微调实现快速适配，显著降低训练成本和数据需求。",
      "rating": 9
    },
    {
      "summary": "批次大小影响训练动态",
      "explanation": "大批次训练稳定但可能陷入尖锐极小值，小批次引入噪声但泛化性能更好。需根据任务权衡。",
      "rating": 7
    }
  ],
  "rating": 8.25,
  "rating_explanation": "该社区代表了现代机器学习训练的核心技术栈，对模型性能和训练效率具有决定性影响。"
}
```

#### 4.3 并发处理所有社区

```python
import concurrent.futures

def generate_community_report(comm_id, entities):
    subgraph = merged_graph.subgraph(entities)
    # ... 提取信息、调用 LLM、返回报告
    return comm_id, report

# 并发生成 6 个社区报告（max_workers=4）
with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
    future_to_comm = {
        executor.submit(generate_community_report, comm_id, entities): comm_id
        for comm_id, entities in communities.items()
    }
    
    reports = {}
    for future in concurrent.futures.as_completed(future_to_comm):
        comm_id, report = future.result()
        reports[comm_id] = report

print(f"生成了 {len(reports)} 个社区报告")
# 输出: 生成了 6 个社区报告
```

#### 4.4 构建社区报告 chunks

```python
community_report_chunks = []

for comm_id, report in reports.items():
    # 获取社区所有文档来源
    doc_ids = set()
    for entity in communities[comm_id]:
        doc_ids.update(merged_graph.nodes[entity]["source_id"])
    
    chunk = {
        "id": f"chunk_community_kb_001_{comm_id}",
        "kb_id": ["kb_001"],
        "knowledge_graph_kwd": "community_report",
        "title_kwd": report["title"],
        "content_with_weight": report["summary"],
        "content_ltks": report["title"] + " " + report["summary"],
        "rank": comm_id,
        "report_json": json.dumps(report, ensure_ascii=False),
        "source_id": list(doc_ids),
        "create_time": "2026-09-18 15:00:00",
        "create_timestamp_flt": time.time()
    }
    community_report_chunks.append(chunk)
```

#### 4.5 批量插入社区报告

```python
# 先删除旧的社区报告
await delete_by_query(
    tenant_id="tenant_xyz",
    kb_id="kb_001",
    knowledge_graph_kwd=["community_report"]
)

# 批量插入新报告
await bulk_insert(community_report_chunks, bulk_size=64)

# 同时更新关系 chunks，添加社区 ID（rank 字段）
for src, tgt in merged_graph.edges():
    src_comm = community_mapping[src]
    tgt_comm = community_mapping[tgt]
    # 如果边连接同一社区内的节点，标记该社区 ID
    if src_comm == tgt_comm:
        edge_chunk["rank"] = src_comm
```

### 输出

**DocStore（Elasticsearch）**：

1. **关系数据**（带社区标签）：
```json
{
  "id": "chunk_graph_kb_001_edge_神经网络_数据集",
  "kb_id": ["kb_001"],
  "knowledge_graph_kwd": "graph",
  "content_with_weight": "神经网络 需要 数据集",
  "source_id": ["doc_A_id_123", "doc_B_id_456"],
  "src_id": "神经网络",
  "tgt_id": "数据集",
  "rank": 0,
  "weight": 9.0
}
```

2. **社区报告**（6 个）：
```json
{
  "id": "chunk_community_kb_001_5",
  "kb_id": ["kb_001"],
  "knowledge_graph_kwd": "community_report",
  "title_kwd": "模型训练优化策略社区",
  "content_with_weight": "该社区聚焦于机器学习模型训练的优化策略...",
  "content_ltks": "模型训练优化策略社区 该社区聚焦于...",
  "rank": 5,
  "report_json": "{\"title\": \"...\", \"summary\": \"...\", \"findings\": [...]}"
}
```

**Redis Phase Marker**：
```bash
graphrag_phase:kb_001 → "PHASE_COMMUNITY"
TTL: 7 days
```

**关键差异点 vs 首次建图**：
- ✅ **重新检测**：整个图重新运行 Leiden，社区划分可能完全改变
- ✅ **新增社区**：doc_B 引入的新主题（优化器、训练策略）形成社区 5
- ✅ **报告更新**：所有社区报告都需要重新生成（即使成员没变，因为关系可能变化）
- ⚠️ **计算成本高**：社区数量增加（5→6），LLM 调用次数增加

---

---

## 最终输出与性能统计

### 数据存储总览

| 存储层 | 键/索引 | 数据类型 | 数量 | 备注 |
|--------|---------|---------|------|------|
| **DocStore** | `knowledge_graph_kwd="graph"` | 节点/边 chunks | 189（69 nodes + 120 edges） | 全局知识图谱 |
| **DocStore** | `knowledge_graph_kwd="subgraph"` | 子图 JSON | 2（doc_A + doc_B） | 按文档存储 |
| **DocStore** | `knowledge_graph_kwd="entity"` | 实体详情 | 69 | 消解后的实体 |
| **DocStore** | `knowledge_graph_kwd="relation"` | 关系详情 | 120 | 带社区标签 |
| **DocStore** | `knowledge_graph_kwd="community_report"` | 社区报告 | 6 | 所有社区更新 |
| **Redis** | `graphrag_phase:kb_001` | Phase marker | 1 | 7天TTL |
| **Redis** | `graphrag_checkpoint:*` | Entity/Community checkpoints | ~100 | 7天TTL |
| **Redis** | `graphrag_llm_cache:*` | LLM 响应缓存 | ~30 | 24h TTL |
| **Redis** | `graphrag_embed_cache:*` | 实体嵌入缓存 | 69 | 24h TTL |

### 性能统计

```json
{
  "ok_docs": ["doc_B_id_456"],
  "failed_docs": [],
  "total_docs": 1,
  "total_chunks": 10,
  "seconds": 245.6,
  "breakdown": {
    "load_chunks": 2.1,
    "generate_subgraph": 85.4,  // LLM 调用主要耗时
    "merge_subgraph": 15.2,
    "resolve_entities": 98.3,   // LLM 判断 + 图操作
    "extract_community": 44.6   // 6 个社区报告生成
  },
  "llm_calls": {
    "subgraph_extraction": 3,   // 3 个批次
    "entity_resolution": 2,     // 12 对 → 2 批次
    "community_reports": 6      // 6 个社区（所有社区都重新生成）
  },
  "cache_hits": {
    "llm_cache": 5,            // 部分实体消解命中缓存
    "embed_cache": 38          // doc_B 的 38 个实体中，5 个命中缓存
  }
}
```

### 与首次建图的对比

| 维度 | 首次建图（doc_A） | 增量追加（doc_B） | 变化 |
|------|------------------|------------------|------|
| **输入节点数** | 0 | 44 | +44 |
| **输出节点数** | 44 | 69 | +25（38 新增 - 5 重叠 - 8 消解） |
| **输出边数** | 78 | 120 | +42 |
| **社区数** | 5 | 6 | +1 |
| **LLM 调用次数** | 9 | 11 | +2（实体消解候选对增加） |
| **总耗时** | 180s | 246s | +36%（图规模增加，社区重新检测） |
| **存储增量** | 200 chunks | +189 chunks | DocStore 数据增加 |

---

## 断点续传示例

### 场景：阶段 3（实体消解）中断

假设在实体消解过程中，处理到第 6 对候选时任务被取消（`task_id=task_999`）。

#### 恢复流程

1. **检查 Phase Marker**：
```python
phase = await redis_client.get("graphrag_phase:kb_001")
# 返回：None 或 "PHASE_MERGE_DONE"（阶段 2 已完成）
```

2. **检查 Entity Checkpoints**：
```python
# 已处理的 5 对实体有 checkpoint
checkpoints = await redis_client.keys("graphrag_checkpoint:entity:kb_001:*")
# 返回：5 个 checkpoints
```

3. **跳过已处理的实体对**：
```python
# entity_resolution.py 内部逻辑
remaining_pairs = similar_pairs[5:]  # 从第 6 对开始
# 继续 LLM 判断和合并
```

4. **完成后更新 Phase Marker**：
```python
await redis_client.setex("graphrag_phase:kb_001", 7*24*3600, "PHASE_RESOLUTION")
```

#### 恢复优势
- ✅ **节省成本**：已完成的 LLM 调用不重复（缓存命中）
- ✅ **快速恢复**：从中断点继续，而非从头开始
- ⚠️ **有效期限制**：7 天内必须恢复，否则 checkpoint 过期

---

## 总结

### 增量文档追加的关键特性

1. **子图生成独立**：新文档的实体关系抽取不依赖现有图谱
2. **智能合并**：重叠实体的 description 和 source_id 自动合并
3. **全局重新检测**：社区划分重新运行，可能产生新社区
4. **存储增量更新**：删除旧 graph/subgraph，插入新数据
5. **缓存复用**：重叠实体的嵌入缓存命中，节省 embedding 成本

### 适用场景

- ✅ 知识库持续扩充（日常添加文档）
- ✅ 多轮迭代优化（添加补充材料）
- ⚠️ 大批量追加（建议批量合并后一次性运行）

### 性能优化建议

- **批量追加**：积累多个文档后一次性运行，减少社区重检测次数
- **缓存预热**：高频实体提前生成嵌入缓存
- **增量社区检测**：未来可优化为仅重新检测受影响的子图（当前全量重检测）
