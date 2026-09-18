# GraphRAG 流程 Demo：首次文档入库

## 场景描述

- **知识库**：kb_001（空库，首次建图）
- **文档**：doc_A.txt，约 5000 字
- **原始分块**：12 个 chunk（已由上游分块模块完成）
- **GraphRAG 方法**：light（LightRAG，默认）
- **实体类型**：["organization", "person", "geo", "event", "category"]
- **语言**：中文

---

## 阶段 0：准备 - 加载与批处理分块

### 输入
- `doc_ids = ["doc_A"]`
- `tenant_id = "tenant_001"`
- `kb_id = "kb_001"`

### 处理逻辑

1. **加载原始分块**
   ```python
   # 从 DocStore 查询
   query = {
       "doc_ids": ["doc_A"],
       "kb_id": "kb_001",
       "fields": ["content_with_weight", "doc_id"]
   }
   chunks = ELASTICSEARCH.search(query, kb_001)
   # 返回 12 个原始 chunk
   ```

2. **Token 计数与批处理**
   - 配置：`batch_chunk_token_size = 4096`（默认值）
   - 对每个 chunk 进行 token 计数（使用 tiktoken）
   - 按 token 限制合并相邻分块

### 输出
- **批次列表**：3 个批次
  - batch_0: chunks[0:5]，约 3800 tokens
  - batch_1: chunks[5:9]，约 3900 tokens
  - batch_2: chunks[9:12]，约 2100 tokens

### Demo 数据示例

**batch_0 示例**
```python
batch_0 = {
    "text": """
    字节跳动成立于2012年，由张一鸣创立，总部位于北京中关村。
    公司旗下拥有抖音、TikTok、今日头条等知名产品。
    抖音是一款短视频社交平台，于2016年9月上线，目前日活用户超过6亿。
    TikTok是抖音的国际版，在全球150多个国家和地区运营。
    张一鸣于2021年宣布卸任CEO，由梁汝波接任。
    """,
    "token_count": 3800,
    "chunk_ids": ["chunk_001", "chunk_002", "chunk_003", "chunk_004", "chunk_005"]
}
```

### 数据存储
无持久化，仅内存中的批次列表传递给下一阶段。

---

## 阶段 1：子图生成 - 实体关系抽取

### 输入
- 3 个文本批次（batch_0, batch_1, batch_2）
- `doc_id = "doc_A"`
- LightKGExt 抽取器实例
- LLM 模型（用于实体关系抽取）
- Embedding 模型（用于向量化）

### 处理逻辑

1. **并发抽取**（每个批次独立处理）
   - 配置：`max_parallel_docs = 4`，本例只有 1 个文档，3 个批次串行处理
   
2. **批次 0 处理流程**
   - **检查点查询**：Redis 查询 `checkpoint:subgraph:{tenant_id}:{kb_id}:{SHA256(batch_0)}`，未命中（首次）
   - **LLM 调用**：
     ```
     System: 你是一个知识图谱抽取助手...
     User: [batch_0 的 5 个 chunk 合并文本]
     ```
   - **重试机制**：最多 2 次重试，指数退避 2.0 秒
   - **超时**：单批次最长 300 秒，最小 600 秒
   - **LLM 缓存**：Redis，key = `llm_cache:{xxHash(prompt)}`，TTL = 24h
   - **取消检查**：每次 LLM 调用前检查 `has_canceled(task_id)`

### Demo 数据示例

#### 2.1 LLM 原始响应（batch_0）

LightRAG 使用分隔符格式，**不是 JSON**。分隔符定义（`rag/graphrag/light/graph_prompt.py`）：
- `tuple_delimiter = "<|>"`
- `record_delimiter = "##"`
- `completion_delimiter = "<|COMPLETE|>"`

```text
("entity"<|>字节跳动<|>ORGANIZATION<|>字节跳动是成立于2012年的中国科技公司，总部位于北京中关村，旗下拥有抖音、TikTok、今日头条等产品。)##
("entity"<|>张一鸣<|>PERSON<|>张一鸣是字节跳动的创始人，于2012年创立公司，2021年宣布卸任CEO。)##
("entity"<|>梁汝波<|>PERSON<|>梁汝波于2021年接任字节跳动CEO职位。)##
("entity"<|>北京<|>GEO<|>北京是字节跳动总部所在城市，具体位于中关村地区。)##
("entity"<|>抖音<|>CATEGORY<|>抖音是字节跳动旗下的短视频社交平台，2016年9月上线，日活用户超过6亿。)##
("entity"<|>TIKTOK<|>CATEGORY<|>TikTok是抖音的国际版本，在全球150多个国家和地区运营。)##
("relationship"<|>张一鸣<|>字节跳动<|>张一鸣于2012年创立了字节跳动，是公司创始人。<|>创始人,创业<|>9)##
("relationship"<|>字节跳动<|>北京<|>字节跳动总部位于北京中关村。<|>总部,地理位置<|>8)##
("relationship"<|>字节跳动<|>抖音<|>抖音是字节跳动开发并运营的短视频产品。<|>产品,开发<|>9)##
("relationship"<|>抖音<|>TIKTOK<|>TikTok是抖音面向海外市场的国际版本。<|>国际化,同源产品<|>8)##
("relationship"<|>梁汝波<|>字节跳动<|>梁汝波于2021年接任字节跳动CEO。<|>任职,管理层<|>8)##
("content_keywords"<|>科技公司,短视频,全球化,管理层变更)
<|COMPLETE|>
```

#### 2.2 解析为节点/边字典（batch_0）

解析逻辑（`handle_single_entity_extraction` / `handle_single_relationship_extraction`）：
- 实体名会被 `.upper()` 统一大写化（注：这里为展示清晰保留原始大小写）
- 边的两端会 `sorted()` 排序保证无向图键一致

```python
# 节点解析结果（maybe_nodes）
{
    "字节跳动": [{
        "entity_name": "字节跳动",
        "entity_type": "ORGANIZATION",
        "description": "字节跳动是成立于2012年的中国科技公司，总部位于北京中关村，旗下拥有抖音、TikTok、今日头条等产品。",
        "source_id": "chunk_001"     # chunk_key，非 doc_id
    }],
    "张一鸣": [{
        "entity_name": "张一鸣",
        "entity_type": "PERSON",
        "description": "张一鸣是字节跳动的创始人，于2012年创立公司，2021年宣布卸任CEO。",
        "source_id": "chunk_001"
    }],
    "梁汝波": [{
        "entity_name": "梁汝波",
        "entity_type": "PERSON",
        "description": "梁汝波于2021年接任字节跳动CEO职位。",
        "source_id": "chunk_002"
    }],
    "北京": [{
        "entity_name": "北京",
        "entity_type": "GEO",
        "description": "北京是字节跳动总部所在城市，具体位于中关村地区。",
        "source_id": "chunk_001"
    }],
    "抖音": [{
        "entity_name": "抖音",
        "entity_type": "CATEGORY",
        "description": "抖音是字节跳动旗下的短视频社交平台，2016年9月上线，日活用户超过6亿。",
        "source_id": "chunk_003"
    }],
    "TIKTOK": [{
        "entity_name": "TIKTOK",
        "entity_type": "CATEGORY",
        "description": "TikTok是抖音的国际版本，在全球150多个国家和地区运营。",
        "source_id": "chunk_004"
    }]
}

# 边解析结果（maybe_edges），键为 sorted 后的元组
{
    ("字节跳动", "张一鸣"): [{
        "src_id": "张一鸣",
        "tgt_id": "字节跳动",
        "weight": 9.0,
        "description": "张一鸣于2012年创立了字节跳动，是公司创始人。",
        "keywords": "创始人,创业",
        "source_id": "chunk_001"
    }],
    ("北京", "字节跳动"): [{
        "src_id": "字节跳动",
        "tgt_id": "北京",
        "weight": 8.0,
        "description": "字节跳动总部位于北京中关村。",
        "keywords": "总部,地理位置",
        "source_id": "chunk_001"
    }],
    ("字节跳动", "抖音"): [{
        "src_id": "字节跳动",
        "tgt_id": "抖音",
        "weight": 9.0,
        "description": "抖音是字节跳动开发并运营的短视频产品。",
        "keywords": "产品,开发",
        "source_id": "chunk_003"
    }],
    ("TIKTOK", "抖音"): [{
        "src_id": "抖音",
        "tgt_id": "TIKTOK",
        "weight": 8.0,
        "description": "TikTok是抖音面向海外市场的国际版本。",
        "keywords": "国际化,同源产品",
        "source_id": "chunk_004"
    }],
    ("字节跳动", "梁汝波"): [{
        "src_id": "梁汝波",
        "tgt_id": "字节跳动",
        "weight": 8.0,
        "description": "梁汝波于2021年接任字节跳动CEO。",
        "keywords": "任职,管理层",
        "source_id": "chunk_002"
    }]
}
```

#### 2.3 构建 NetworkX 子图（batch_0）

```python
import networkx as nx

subgraph_0 = nx.Graph()

# 添加所有节点（6 个实体）
subgraph_0.add_node("字节跳动",
    entity_type="ORGANIZATION",
    description="字节跳动是成立于2012年的中国科技公司，总部位于北京中关村，旗下拥有抖音、TikTok、今日头条等产品。",
    source_id=["chunk_001"])

subgraph_0.add_node("张一鸣",
    entity_type="PERSON",
    description="张一鸣是字节跳动的创始人，于2012年创立公司，2021年宣布卸任CEO。",
    source_id=["chunk_001"])

subgraph_0.add_node("梁汝波",
    entity_type="PERSON",
    description="梁汝波于2021年接任字节跳动CEO职位。",
    source_id=["chunk_002"])

subgraph_0.add_node("北京",
    entity_type="GEO",
    description="北京是字节跳动总部所在城市，具体位于中关村地区。",
    source_id=["chunk_001"])

subgraph_0.add_node("抖音",
    entity_type="CATEGORY",
    description="抖音是字节跳动旗下的短视频社交平台，2016年9月上线，日活用户超过6亿。",
    source_id=["chunk_003"])

subgraph_0.add_node("TIKTOK",
    entity_type="CATEGORY",
    description="TikTok是抖音的国际版本，在全球150多个国家和地区运营。",
    source_id=["chunk_004"])

# 添加所有边（5 条关系）
subgraph_0.add_edge("字节跳动", "张一鸣",
    description="张一鸣于2012年创立了字节跳动，是公司创始人。",
    keywords=["创始人", "创业"],
    weight=9.0,
    source_id=["chunk_001"])

subgraph_0.add_edge("字节跳动", "北京",
    description="字节跳动总部位于北京中关村。",
    keywords=["总部", "地理位置"],
    weight=8.0,
    source_id=["chunk_001"])

subgraph_0.add_edge("字节跳动", "抖音",
    description="抖音是字节跳动开发并运营的短视频产品。",
    keywords=["产品", "开发"],
    weight=9.0,
    source_id=["chunk_003"])

subgraph_0.add_edge("抖音", "TIKTOK",
    description="TikTok是抖音面向海外市场的国际版本。",
    keywords=["国际化", "同源产品"],
    weight=8.0,
    source_id=["chunk_004"])

subgraph_0.add_edge("字节跳动", "梁汝波",
    description="梁汝波于2021年接任字节跳动CEO。",
    keywords=["任职", "管理层"],
    weight=8.0,
    source_id=["chunk_002"])

# 统计信息
print(f"Subgraph 0: {subgraph_0.number_of_nodes()} 节点, {subgraph_0.number_of_edges()} 边")
# => Subgraph 0: 6 节点, 5 边
```

#### 2.4 多次抽取的描述拼接示例

当 batch_0 和 batch_1 都抽取到同一实体（如"字节跳动"）时，使用 `GRAPH_FIELD_SEP = "<SEP>"` 拼接描述：

```python
# 假设 batch_1 也抽取到 "字节跳动"，新增描述：
# "字节跳动2021年营收达到617亿美元，员工超过10万人。"

# 合并后的节点属性：
subgraph.nodes["字节跳动"]["description"]
# => "字节跳动是成立于2012年的中国科技公司，总部位于北京中关村，旗下拥有抖音、TikTok、今日头条等产品。<SEP>字节跳动2021年营收达到617亿美元，员工超过10万人。"

subgraph.nodes["字节跳动"]["source_id"]
# => ["chunk_001", "chunk_007"]  # 来自两个不同批次的 chunk
```

#### 2.5 重复处理 batch_1, batch_2

- batch_1 处理：生成 subgraph_1（假设 8 节点，12 边）
- batch_2 处理：生成 subgraph_2（假设 7 节点，10 边）

#### 2.6 合并文档级子图

```python
from rag.graphrag.utils import graph_merge

doc_subgraph = nx.Graph()

# 合并 3 个批次的子图
for sg in [subgraph_0, subgraph_1, subgraph_2]:
    doc_subgraph = graph_merge(doc_subgraph, sg)

# 合并逻辑（graph_merge 函数）：
# 1. 节点合并：如果节点已存在，累加 description（用 <SEP> 分隔），合并 source_id
# 2. 边合并：如果边已存在，累加 description 和 weight

# 最终文档子图统计
print(f"Doc subgraph: {doc_subgraph.number_of_nodes()} 节点, {doc_subgraph.number_of_edges()} 边")
# => Doc subgraph: 45 节点, 78 边（去重后）
```

### 输出
- **文档子图**：NetworkX Graph 对象
  - 节点数：45 个实体
  - 边数：78 条关系
  - 节点属性：entity_type, description, source_id
  - 边属性：description, weight, keywords, source_id

### 数据存储

#### DocStore (Elasticsearch) - 子图缓存

```python
import json

chunk = {
    "id": "subgraph_doc_A_uuid",
    "content_with_weight": json.dumps(nx.node_link_data(doc_subgraph, edges="edges"), ensure_ascii=False),
    "knowledge_graph_kwd": "subgraph",  # 标识为子图
    "kb_id": "kb_001",
    "source_id": ["doc_A"],  # 关联文档
    "create_time": "2026-09-18 10:23:45",
    "create_timestamp_flt": 1726628625.0,
    "available_int": 0,
    "removed_kwd": "N"
}
ELASTICSEARCH.insert([chunk], "kb_001")
```

示例 `content_with_weight` 内容（部分）：
```json
{
  "directed": false,
  "multigraph": false,
  "graph": {"source_id": ["doc_A"]},
  "nodes": [
    {
      "id": "字节跳动",
      "entity_type": "ORGANIZATION",
      "description": "字节跳动是成立于2012年的中国科技公司...",
      "source_id": ["chunk_001", "chunk_007"]
    },
    {
      "id": "张一鸣",
      "entity_type": "PERSON",
      "description": "张一鸣是字节跳动的创始人...",
      "source_id": ["chunk_001"]
    }
  ],
  "edges": [
    {
      "source": "字节跳动",
      "target": "张一鸣",
      "description": "张一鸣于2012年创立了字节跳动，是公司创始人。",
      "keywords": ["创始人", "创业"],
      "weight": 9.0,
      "source_id": ["chunk_001"]
    }
  ]
}
```

#### Redis - 批次级检查点（用于断点续传）

3 个批次分别保存检查点：

```python
import hashlib
import json

# batch_0 检查点
batch_0_hash = hashlib.sha256(batch_0["text"].encode()).hexdigest()
key_0 = f"checkpoint:subgraph:tenant_001:kb_001:{batch_0_hash}"
value_0 = json.dumps({
    "entities": [
        {"entity_name": "字节跳动", "entity_type": "ORGANIZATION", "description": "..."},
        {"entity_name": "张一鸣", "entity_type": "PERSON", "description": "..."},
        # ... 其他实体
    ],
    "relationships": [
        {"src_id": "张一鸣", "tgt_id": "字节跳动", "weight": 9.0, "description": "..."},
        # ... 其他关系
    ]
}, ensure_ascii=False)

redis.set(key_0, value_0)
redis.expire(key_0, 7 * 86400)  # 7 天 TTL

# batch_1 和 batch_2 同理
```

---

## 阶段 2：子图合并 - 构建全局知识图谱

### 输入
- 文档子图（doc_subgraph）：45 节点，78 边
- `doc_id = "doc_A"`
- 全局图查询结果：空（kb_001 首次建图）

### 处理逻辑

1. **查询现有全局图**
   ```python
   # 从 DocStore 查询
   query = {
       "kb_id": "kb_001",
       "knowledge_graph_kwd": ["graph"]
   }
   result = ELASTICSEARCH.search(query)
   # 返回：空（首次建图）
   global_graph = nx.Graph()
   ```

2. **图合并算法**（`graph_merge` 函数）
   ```python
   # 由于全局图为空，直接复制子图
   for node, attrs in doc_subgraph.nodes(data=True):
       global_graph.add_node(node, **attrs)
   
   for u, v, attrs in doc_subgraph.edges(data=True):
       global_graph.add_edge(u, v, **attrs)
   
   global_graph.graph["source_id"] = ["doc_A"]
   ```

3. **记录变更**（用于后续增量更新）
   ```python
   from rag.graphrag.utils import GraphChange
   
   change = GraphChange(
       removed_nodes=set(),
       added_updated_nodes={"字节跳动", "张一鸣", "北京", ...},  # 45 个节点
       removed_edges=set(),
       added_updated_edges={("张一鸣", "字节跳动"), ("字节跳动", "北京"), ...}  # 78 条边
   )
   ```

4. **调用 `set_graph` 持久化**
   - 构建全局图 chunk
   - 构建子图 chunk（按 doc_id）
   - 为每个节点生成嵌入向量
   - 为每条边生成嵌入向量
   - 批量插入 DocStore

### Demo 数据示例

#### 3.1 全局图与变更对象

```python
# 首次建图，全局图直接复制文档子图
global_graph = doc_subgraph.copy()
global_graph.graph["source_id"] = ["doc_A"]

print(f"Global graph: {global_graph.number_of_nodes()} 节点, {global_graph.number_of_edges()} 边")
# => Global graph: 45 节点, 78 边

# 变更对象（记录所有新增）
change = GraphChange(
    removed_nodes=set(),  # 无删除
    added_updated_nodes={
        "字节跳动", "张一鸣", "梁汝波", "北京", "抖音", "TIKTOK",
        # ... 其余 39 个实体
    },
    removed_edges=set(),  # 无删除
    added_updated_edges={
        ("字节跳动", "张一鸣"), ("字节跳动", "北京"), ("字节跳动", "抖音"),
        ("抖音", "TIKTOK"), ("字节跳动", "梁汝波"),
        # ... 其余 73 条边
    }
)
```

#### 3.2 生成实体和关系 chunk

调用 `set_graph` 函数，为每个节点和边生成 chunk 并计算嵌入向量。

**实体 chunk 示例**（2 个）：

```python
# 实体 1: 字节跳动
entity_chunk_1 = {
    "id": "entity_字节跳动_uuid",
    "entity_kwd": "字节跳动",
    "important_kwd": ["字节跳动"],
    "knowledge_graph_kwd": "entity",
    "entity_type_kwd": "ORGANIZATION",
    "content_with_weight": json.dumps({
        "entity_type": "ORGANIZATION",
        "description": "字节跳动是成立于2012年的中国科技公司，总部位于北京中关村，旗下拥有抖音、TikTok、今日头条等产品。<SEP>字节跳动2021年营收达到617亿美元，员工超过10万人。",
        "source_id": ["chunk_001", "chunk_007"]
    }, ensure_ascii=False),
    "content_ltks": ["字节", "跳动", "2012", "科技", "公司", ...],  # 分词结果
    "source_id": ["chunk_001", "chunk_007"],
    "rank_flt": 0.0,  # PageRank 初始值（社区检测后更新）
    "n_hop_with_weight": json.dumps([], ensure_ascii=False),  # N-hop 邻居（社区检测后更新）
    "q_1024_vec": [0.123, -0.456, 0.789, ...],  # 1024 维嵌入向量
    "kb_id": "kb_001",
    "available_int": 0
}

# 实体 2: 张一鸣
entity_chunk_2 = {
    "id": "entity_张一鸣_uuid",
    "entity_kwd": "张一鸣",
    "important_kwd": ["张一鸣"],
    "knowledge_graph_kwd": "entity",
    "entity_type_kwd": "PERSON",
    "content_with_weight": json.dumps({
        "entity_type": "PERSON",
        "description": "张一鸣是字节跳动的创始人，于2012年创立公司，2021年宣布卸任CEO。",
        "source_id": ["chunk_001"]
    }, ensure_ascii=False),
    "content_ltks": ["张一鸣", "字节", "跳动", "创始人", "2012", ...],
    "source_id": ["chunk_001"],
    "rank_flt": 0.0,
    "n_hop_with_weight": json.dumps([], ensure_ascii=False),
    "q_1024_vec": [0.234, 0.567, -0.123, ...],
    "kb_id": "kb_001",
    "available_int": 0
}
```

**关系 chunk 示例**（1 个）：

```python
# 关系: 张一鸣 -> 字节跳动
relation_chunk = {
    "id": "relation_张一鸣_字节跳动_uuid",
    "from_entity_kwd": "张一鸣",
    "to_entity_kwd": "字节跳动",
    "knowledge_graph_kwd": "relation",
    "content_with_weight": json.dumps({
        "description": "张一鸣于2012年创立了字节跳动，是公司创始人。",
        "weight": 9.0,
        "keywords": "创始人,创业",
        "source_id": ["chunk_001"]
    }, ensure_ascii=False),
    "content_ltks": ["张一鸣", "2012", "创立", "字节", "跳动", "创始人"],
    "important_kwd": ["创始人", "创业"],
    "source_id": ["chunk_001"],
    "weight_int": 9,
    "q_1024_vec": [0.345, -0.678, 0.912, ...],
    "kb_id": "kb_001",
    "available_int": 0
}
```

#### 3.3 嵌入向量生成与缓存

```python
# 批量预热（Batch Pre-warm）策略
# 1. 收集所有需要嵌入的文本（45 实体 + 78 边 = 123 个）
texts_to_embed = []
for node in global_graph.nodes():
    texts_to_embed.append(global_graph.nodes[node]["description"])
for u, v in global_graph.edges():
    texts_to_embed.append(global_graph[u][v]["description"])

# 2. 批量检查 Redis 缓存
cache_keys = [f"embed_cache:{xxhash.xxh64((embedding_model_name + text).encode()).hexdigest()}" 
              for text in texts_to_embed]
cached_vectors = redis.mget(cache_keys)  # 批量查询

# 3. 只为缓存未命中的文本调用嵌入模型
uncached_indices = [i for i, vec in enumerate(cached_vectors) if vec is None]
print(f"缓存命中: {len(cached_vectors) - len(uncached_indices)}/123")
# => 缓存命中: 0/123（首次）

# 4. 批量生成嵌入（减少 API 调用）
uncached_texts = [texts_to_embed[i] for i in uncached_indices]
new_vectors = embedding_model.encode(uncached_texts)  # 一次调用生成 123 个向量

# 5. 批量写入 Redis 缓存
for i, vec in zip(uncached_indices, new_vectors):
    redis.set(cache_keys[i], json.dumps(vec.tolist()), ex=86400)  # 24h TTL
```

### 输出
- **全局图**：NetworkX Graph 对象
  - 节点数：45
  - 边数：78
  - 图属性：source_id = ["doc_A"]
- **变更对象**：GraphChange（记录所有新增节点和边）

### 数据存储

#### DocStore (Elasticsearch) - 全局图

```python
chunk = {
    "id": "graph_kb_001_uuid",
    "content_with_weight": json.dumps(nx.node_link_data(global_graph, edges="edges"), ensure_ascii=False),
    "knowledge_graph_kwd": "graph",  # 标识为全局图
    "kb_id": "kb_001",
    "source_id": ["doc_A"],
    "create_time": "2026-09-18 10:24:10",
    "create_timestamp_flt": 1726628650.0,
    "available_int": 0,
    "removed_kwd": "N"
}
ELASTICSEARCH.insert([chunk], "kb_001")
```

#### DocStore (Elasticsearch) - 实体节点（45 个）

批量插入配置：
- 批量大小：64 个 chunk/批次
- 并发度：4 个批次并行
- 重试：每批次 3 次，指数退避

```python
entity_chunks = [entity_chunk_1, entity_chunk_2, ...]  # 45 个实体 chunk
ELASTICSEARCH.insert(entity_chunks, "kb_001", bulk_size=64)
```

#### DocStore (Elasticsearch) - 关系边（78 条）

```python
relation_chunks = [relation_chunk, ...]  # 78 个关系 chunk
ELASTICSEARCH.insert(relation_chunks, "kb_001", bulk_size=64)
```

#### Redis - 嵌入缓存（123 个）

实体和边的向量，加速后续查询：

```python
# 示例：字节跳动的嵌入缓存
key = f"embed_cache:{xxhash.xxh64(('bge-large-zh-v1.5' + description_text).encode()).hexdigest()}"
value = json.dumps([0.123, -0.456, 0.789, ...])  # 1024 维向量
redis.set(key, value, ex=86400)  # TTL: 24h
```

---

## 阶段 3：实体消解 - 合并相似实体

### 输入
- 全局图（45 节点，78 边）
- 新增节点集合：{"字节跳动", "张一鸣", "北京", ...}（45 个）
- EntityResolution 抽取器实例
- LLM 模型（用于语义判断）

### 处理逻辑

1. **查询 Phase Marker**
   ```python
   # Redis 查询
   key = f"phase_marker:tenant_001:kb_001"
   phase = redis.get(key)  # 返回 None（首次）
   # 由于 with_resolution=True，执行实体消解
   ```

2. **加载检查点**
   ```python
   # Redis 查询已消解的实体对
   checkpoint_key = "checkpoint:resolution:tenant_001:kb_001"
   checkpoints = redis.hgetall(checkpoint_key)  # 返回 {}（首次）
   ```

3. **生成候选实体对**（使用 rapidfuzz）
   ```python
   import itertools
   from rapidfuzz import fuzz
   
   # 预过滤：Levenshtein 相似度 > 0.8
   candidates = []
   for e1, e2 in itertools.combinations(new_nodes, 2):
       if fuzz.ratio(e1, e2) > 80:
           candidates.append((e1, e2))
   
   # 假设找到 2 对候选
   candidates = [("字节跳动", "ByteDance"), ("张一鸣", "张一明")]
   print(f"候选实体对: {len(candidates)}")
   # => 候选实体对: 2
   ```

### Demo 数据示例

#### 4.1 LLM 判断批次

批次配置：
- 批次大小：100 对/批次
- 并发度：5 个批次并行

**批次 0**（2 对候选）：

```python
# 1. 构建 LLM Prompt
prompt = """
判断以下实体对是否指向同一实体。对每一对，基于上下文信息判断是否为同一实体。

实体对 1:
- 实体 A: "字节跳动"
  描述: 字节跳动是成立于2012年的中国科技公司，总部位于北京中关村，旗下拥有抖音、TikTok、今日头条等产品。<SEP>字节跳动2021年营收达到617亿美元，员工超过10万人。
- 实体 B: "BYTEDANCE"
  描述: ByteDance is a Chinese technology company founded in 2012, headquartered in Beijing.

实体对 2:
- 实体 A: "张一鸣"
  描述: 张一鸣是字节跳动的创始人，于2012年创立公司，2021年宣布卸任CEO。
- 实体 B: "张一明"
  描述: 张一明是一位软件工程师。

请以 JSON 格式返回判断结果：
{
  "resolutions": [
    {"pair": 0, "same": true/false, "reason": "..."},
    {"pair": 1, "same": true/false, "reason": "..."}
  ]
}
"""

# 2. LLM 调用
response = llm_model.chat(prompt)

# 3. LLM 响应
{
    "resolutions": [
        {
            "pair": 0,
            "same": true,
            "reason": "两者都指向同一家公司，只是中英文名称不同。实体描述内容一致：都是2012年成立的中国科技公司，总部在北京。"
        },
        {
            "pair": 1,
            "same": false,
            "reason": "虽然姓名相似（可能是同音或笔误），但描述内容完全不同。张一鸣是字节跳动创始人，而张一明是普通软件工程师，显然是两个不同的人。"
        }
    ]
}
```

#### 4.2 执行图合并

基于 LLM 判断结果，只合并实体对 0（same=true）：

```python
# 1. 对于实体对 0: "字节跳动" + "BYTEDANCE"
source_entity = "字节跳动"
target_entity = "BYTEDANCE"

# 2. 获取两个节点的数据
source_data = global_graph.nodes[source_entity]
target_data = global_graph.nodes[target_entity]

# 3. 合并 description（使用 GRAPH_FIELD_SEP）
merged_description = source_data["description"] + GRAPH_FIELD_SEP + target_data["description"]
# => "字节跳动是成立于2012年的中国科技公司，总部位于北京中关村，旗下拥有抖音、TikTok、今日头条等产品。<SEP>字节跳动2021年营收达到617亿美元，员工超过10万人。<SEP>ByteDance is a Chinese technology company founded in 2012, headquartered in Beijing."

# 4. 合并 source_id
merged_source_id = list(set(source_data["source_id"] + target_data.get("source_id", [])))
# => ["doc_A"]

# 5. 更新源节点
global_graph.nodes[source_entity]["description"] = merged_description
global_graph.nodes[source_entity]["source_id"] = merged_source_id

# 6. 重定向所有指向 "BYTEDANCE" 的边到 "字节跳动"
for neighbor in list(global_graph.neighbors(target_entity)):
    if global_graph.has_edge(source_entity, neighbor):
        # 边已存在，合并边属性
        edge_data_old = global_graph[source_entity][neighbor]
        edge_data_new = global_graph[target_entity][neighbor]
        edge_data_old["description"] += GRAPH_FIELD_SEP + edge_data_new["description"]
        edge_data_old["weight"] = max(edge_data_old["weight"], edge_data_new["weight"])
    else:
        # 创建新边
        edge_data = global_graph[target_entity][neighbor]
        global_graph.add_edge(source_entity, neighbor, **edge_data)
    # 删除旧边
    global_graph.remove_edge(target_entity, neighbor)

# 7. 删除目标节点
global_graph.remove_node(target_entity)

# 8. 记录变更
change.removed_nodes.add("BYTEDANCE")
change.added_updated_nodes.add("字节跳动")

print(f"合并完成: 节点数 {global_graph.number_of_nodes()}, 边数 {global_graph.number_of_edges()}")
# => 合并完成: 节点数 44, 边数 78
```

#### 4.3 保存检查点

每处理完一对实体，立即保存检查点以支持断点续传：

```python
import hashlib

# 对于实体对 0
checkpoint_key = "checkpoint:resolution:tenant_001:kb_001"
pair_key = "字节跳动" + GRAPH_FIELD_SEP + "BYTEDANCE"
field = hashlib.sha256(pair_key.encode()).hexdigest()

checkpoint_value = {
    "same": True,
    "kept": "字节跳动",
    "removed": "BYTEDANCE",
    "timestamp": "2026-09-18 10:24:45"
}

redis.hset(checkpoint_key, field, json.dumps(checkpoint_value))
redis.expire(checkpoint_key, 7 * 86400)  # 7 天 TTL

# 对于实体对 1（same=false，也保存以避免重复判断）
pair_key_2 = "张一鸣" + GRAPH_FIELD_SEP + "张一明"
field_2 = hashlib.sha256(pair_key_2.encode()).hexdigest()

checkpoint_value_2 = {
    "same": False,
    "reason": "虽然姓名相似但人物不同",
    "timestamp": "2026-09-18 10:24:45"
}

redis.hset(checkpoint_key, field_2, json.dumps(checkpoint_value_2))
```

#### 4.4 更新存储

调用 `set_graph` 将合并后的图持久化到 DocStore：

```python
from rag.graphrag.utils import set_graph

# 批量更新实体、关系、全局图
set_graph(
    tenant_id="tenant_001",
    kb_id="kb_001",
    graph=global_graph,
    change=change,
    doc_store=ELASTICSEARCH,
    embedding_model=embedding_model,
    redis_client=redis
)
```

这会执行以下操作：
1. **删除** "BYTEDANCE" 实体 chunk（根据 `change.removed_nodes`）
2. **更新** "字节跳动" 实体 chunk（重新计算嵌入向量，因为 description 改变了）
3. **更新**所有相关的关系边 chunk
4. **更新**全局图 chunk（节点数从 45 变为 44）

### 输出
- **更新后的全局图**：44 节点，78 边
  - 合并了 1 对实体（"字节跳动" + "ByteDance"）
- **变更对象**：GraphChange
  - removed_nodes: {"ByteDance"}
  - added_updated_nodes: {"字节跳动"}

### 数据存储

**Redis - Phase Marker**
```python
# 标记实体消解阶段完成
key = "phase_marker:tenant_001:kb_001"
redis.set(key, "PHASE_RESOLUTION")
redis.expire(key, 7 * 86400)  # 7 天 TTL
```

**Redis - 消解检查点**
- Key: `checkpoint:resolution:tenant_001:kb_001`
- Type: Hash
- Fields: SHA256 哈希的实体对
- Values: `{"same": true/false, "kept": "...", "removed": "..."}`
- TTL: 7 天

**DocStore (Elasticsearch) - 更新全局图和实体**
- 删除旧的 "ByteDance" 实体 chunk
- 更新 "字节跳动" 实体 chunk（合并后的 description）
- 更新全局图 chunk（节点数 44）
- 更新所有相关的关系边 chunk

**超时配置**
- 总超时：1800 秒（30 分钟）

---

## 阶段 4：社区检测 - 生成社区报告

### 输入
- 更新后的全局图（44 节点，78 边）
- CommunityReportsExtractor 抽取器实例
- LLM 模型（用于生成报告）

### 处理逻辑

1. **查询 Phase Marker**
2. **Leiden 社区检测**
3. **为每个社区构建子图**
4. **加载检查点**
5. **生成社区报告**（并发处理）
6. **保存检查点**
7. **构建社区报告 chunks**
8. **批量插入社区报告**

### Demo 数据示例

#### 5.1 查询 Phase Marker

```python
# Redis 查询
key = "phase_marker:tenant_001:kb_001"
phase = redis.get(key)
print(f"当前阶段: {phase}")
# => 当前阶段: PHASE_RESOLUTION

# 由于 with_community=True，继续执行社区检测
```

#### 5.2 Leiden 社区检测

```python
from rag.graphrag.general.leiden import run as leiden_run

# 运行 Leiden 算法
community_mapping = leiden_run(global_graph, {})

# 返回：{节点名: 社区ID}
print(f"检测到 {len(set(community_mapping.values()))} 个社区")
# => 检测到 5 个社区

# 示例输出（部分）
{
    "字节跳动": 0,
    "张一鸣": 0,
    "抖音": 0,
    "TIKTOK": 0,
    "今日头条": 0,
    "梁汝波": 0,
    "张利东": 0,
    "ALGORITHM": 0,
    "推荐系统": 0,
    "AI技术": 0,
    "内容分发": 0,
    "DAU": 0,
    "北京": 1,
    "中关村": 1,
    "海淀区": 1,
    "上地": 1,
    "上海": 2,
    "深圳": 2,
    ...
}

# 按社区ID分组实体
from collections import defaultdict
communities = defaultdict(list)
for entity, comm_id in community_mapping.items():
    communities[comm_id].append(entity)

# 社区分布
for comm_id, entities in sorted(communities.items()):
    print(f"社区 {comm_id}: {len(entities)} 个实体")
# => 社区 0: 12 个实体  （字节跳动生态）
# => 社区 1: 8 个实体   （北京地理）
# => 社区 2: 10 个实体  （其他城市）
# => 社区 3: 9 个实体   （技术栈）
# => 社区 4: 5 个实体   （竞争对手）
```

#### 5.3 为社区 0 构建子图

```python
# 社区 0: 字节跳动生态
community_0_entities = [
    "字节跳动", "张一鸣", "抖音", "TIKTOK", "今日头条",
    "梁汝波", "张利东", "ALGORITHM", "推荐系统", "AI技术", "内容分发", "DAU"
]

# 构建子图（只包含社区内部的节点和边）
community_0_graph = global_graph.subgraph(community_0_entities)

print(f"社区 0 子图: {community_0_graph.number_of_nodes()} 节点, {community_0_graph.number_of_edges()} 边")
# => 社区 0 子图: 12 节点, 18 边
```

#### 5.4 加载检查点

```python
# Redis 查询已生成的社区报告
checkpoint_key = "checkpoint:community:tenant_001:kb_001"
checkpoints = redis.hgetall(checkpoint_key)
print(f"已完成的社区: {len(checkpoints)}")
# => 已完成的社区: 0（首次运行）
```

#### 5.5 生成社区 0 的报告

**步骤 1：提取实体和关系信息**

```python
# 1. 提取实体信息
entities_info = []
for entity in community_0_entities:
    node_data = community_0_graph.nodes[entity]
    entities_info.append({
        "name": entity,
        "type": node_data.get("entity_type", "unknown"),
        "description": node_data.get("description", "")[:100]  # 截断长描述
    })

# 2. 提取关系信息
relationships_info = []
for u, v in community_0_graph.edges():
    edge_data = community_0_graph[u][v]
    relationships_info.append({
        "source": u,
        "target": v,
        "description": edge_data.get("description", "")[:80],
        "weight": edge_data.get("weight", 0)
    })

# 3. 按权重排序（取前10条最重要的关系）
relationships_info.sort(key=lambda x: x["weight"], reverse=True)
top_relationships = relationships_info[:10]
```

**步骤 2：构建 LLM Prompt**

```python
# 格式化实体信息
entities_text = "\n".join([
    f"{i+1}. {e['name']} ({e['type']}): {e['description']}"
    for i, e in enumerate(entities_info)
])

# 格式化关系信息
relationships_text = "\n".join([
    f"- {r['source']} -> {r['target']}: {r['description']} (权重: {r['weight']:.2f})"
    for r in top_relationships
])

prompt = f"""
基于以下实体和关系，生成一份结构化的社区分析报告。

实体列表（{len(entities_info)} 个）：
{entities_text}

关系列表（前 10 条）：
{relationships_text}

请以 JSON 格式返回报告：
{{
  "title": "社区标题（20字以内）",
  "summary": "社区摘要（100-200字，概述该社区的核心主题和重要性）",
  "findings": [
    {{
      "summary": "发现标题",
      "explanation": "详细说明（100-200字）",
      "rating": 重要性评分（0-10）
    }}
  ],
  "rating": 社区整体重要性（0-10），
  "rating_explanation": "评分理由（50-100字）"
}}

要求：
- findings 至少 3 条，按重要性排序
- 关注实体之间的连接和影响力
- 评分应基于实体数量、连接密度和领域重要性
"""
```

**步骤 3：LLM 调用与响应**

```python
# LLM 调用
response = llm_model.chat(prompt)

# LLM 响应示例
community_0_report = {
    "title": "字节跳动科技生态圈",
    "summary": "该社区以字节跳动公司为核心，包含其创始人张一鸣、现任CEO梁汝波等关键人物，以及抖音、TikTok、今日头条等核心产品。社区展现了字节跳动在推荐算法、AI技术和内容分发领域的技术优势，以及通过产品矩阵构建的全球化内容生态系统。",
    "findings": [
        {
            "summary": "全球化内容平台布局",
            "explanation": "字节跳动通过抖音（国内）和TikTok（海外）形成双轮驱动，配合今日头条的资讯分发，构建了覆盖短视频、长视频、图文资讯的完整内容生态。TikTok的全球DAU超过10亿，成为中国互联网公司出海的标杆案例。",
            "rating": 9
        },
        {
            "summary": "推荐算法核心竞争力",
            "explanation": "社区中多次强调Algorithm（推荐系统）和AI技术的核心地位。字节跳动的推荐算法被认为是其产品成功的关键因素，通过精准的内容分发提升用户留存和DAU指标，形成了独特的技术壁垒。",
            "rating": 9
        },
        {
            "summary": "创始团队与管理层传承",
            "explanation": "张一鸣作为创始人建立了公司文化和技术基因，2021年卸任CEO后由梁汝波接任。张利东等核心管理层的稳定性保证了公司战略的连续性。这种人才梯队建设是企业持续发展的重要保障。",
            "rating": 7
        },
        {
            "summary": "内容分发的技术创新",
            "explanation": "通过机器学习和大数据分析，字节跳动在内容分发领域实现了从'人找内容'到'内容找人'的范式转变。这一创新模式被众多竞争对手效仿，深刻影响了整个互联网内容行业。",
            "rating": 8
        }
    ],
    "rating": 8.5,
    "rating_explanation": "该社区代表了中国互联网科技的重要力量，在全球范围内具有显著影响力。其技术创新、产品矩阵和全球化战略值得深入研究。"
}
```

#### 5.6 保存社区 0 的检查点

```python
import hashlib

checkpoint_key = "checkpoint:community:tenant_001:kb_001"

# 使用社区实体列表的 SHA256 哈希作为 field
entities_sorted = sorted(community_0_entities)
field = hashlib.sha256(json.dumps(entities_sorted).encode()).hexdigest()

# 保存完整报告
redis.hset(checkpoint_key, field, json.dumps(community_0_report, ensure_ascii=False))
redis.expire(checkpoint_key, 7 * 86400)  # 7 天 TTL

print(f"社区 0 检查点已保存: {field[:16]}...")
# => 社区 0 检查点已保存: a7f3c9e1b2d4f5a6...
```

#### 5.7 并发处理其余社区

```python
import concurrent.futures

# 社区 1-4 并发生成报告
remaining_communities = {
    1: ["北京", "中关村", "海淀区", "上地", ...],
    2: ["上海", "深圳", ...],
    3: ["PYTHON", "GO", "REACT", ...],
    4: ["腾讯", "阿里巴巴", ...]
}

with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
    future_to_comm = {
        executor.submit(generate_community_report, comm_id, entities): comm_id
        for comm_id, entities in remaining_communities.items()
    }
    
    for future in concurrent.futures.as_completed(future_to_comm):
        comm_id = future_to_comm[future]
        report = future.result()
        print(f"社区 {comm_id} 报告生成完成: {report['title']}")

# 输出：
# => 社区 1 报告生成完成: 北京科技园区地理分布
# => 社区 2 报告生成完成: 一线城市互联网布局
# => 社区 3 报告生成完成: 技术栈与开发生态
# => 社区 4 报告生成完成: 竞争格局与市场态势
```

#### 5.8 构建社区报告 chunks

```python
all_reports = [community_0_report, ...]  # 5 个报告

community_report_chunks = []

for comm_id, report in enumerate(all_reports):
    # 获取该社区关联的文档（通过实体的 source_id）
    doc_ids = set()
    for entity in communities[comm_id]:
        entity_source_ids = global_graph.nodes[entity].get("source_id", [])
        doc_ids.update(entity_source_ids)
    
    chunk = {
        "docnm_kwd": report["title"],
        "content_with_weight": json.dumps(report, ensure_ascii=False),
        "knowledge_graph_kwd": "community_report",
        "entities_kwd": communities[comm_id],  # 该社区的所有实体
        "kb_id": "kb_001",
        "source_id": list(doc_ids),
        "create_time": "2026-09-18 10:25:30",
        "create_timestamp_flt": 1726628730.0
    }
    community_report_chunks.append(chunk)

print(f"生成 {len(community_report_chunks)} 个社区报告 chunks")
# => 生成 5 个社区报告 chunks
```

#### 5.9 批量插入社区报告

```python
# 批量插入到 Elasticsearch
ELASTICSEARCH.insert(community_report_chunks, "kb_001", bulk_size=64)

print("社区报告已全部插入 DocStore")
```

### 输出
- **社区报告列表**：5 个报告
  - 每个报告包含：title, summary, findings, rating, entities
- **社区分配**：所有节点的 community_id 属性更新

### 数据存储

**DocStore (Elasticsearch) - 社区报告**（5 个，举例 1 个）
```python
chunk = {
    "docnm_kwd": "字节跳动科技生态圈",
    "content_with_weight": json.dumps({
        "title": "字节跳动科技生态圈",
        "summary": "该社区围绕字节跳动及其创始人张一鸣...",
        "findings": [...],
        "rating": 8.5,
        "rating_explanation": "..."
    }, ensure_ascii=False),
    "knowledge_graph_kwd": "community_report",
    "entities_kwd": ["字节跳动", "张一鸣", "抖音", "TikTok", ...],
    "kb_id": "kb_001",
    "source_id": ["doc_A"],
    "create_time": "2026-09-18 10:25:30",
    "create_timestamp_flt": 1726628730.0
}
```

**Redis - Phase Marker**
```python
# 标记社区检测阶段完成
key = "phase_marker:tenant_001:kb_001"
redis.set(key, "PHASE_COMMUNITY")
redis.expire(key, 7 * 86400)  # 7 天 TTL
```

**Redis - 社区检查点**
- Key: `checkpoint:community:tenant_001:kb_001`
- Type: Hash
- Fields: SHA256 哈希的实体列表
- Values: 完整的社区报告 JSON
- TTL: 7 天

**超时配置**
- 总超时：1800 秒（30 分钟）

---

## 最终输出与性能统计

### 函数返回值
```python
result = {
    "ok_docs": ["doc_A"],           # 成功处理的文档
    "failed_docs": [],              # 失败的文档
    "total_docs": 1,                # 总文档数
    "total_chunks": 12,             # 总分块数
    "seconds": 245.6                # 总耗时（秒）
}
```

### 存储总览

| 数据类型 | 存储位置 | 数量 | 标识字段 |
|---------|---------|------|---------|
| 全局图 | DocStore | 1 | `knowledge_graph_kwd="graph"` |
| 文档子图 | DocStore | 1 | `knowledge_graph_kwd="subgraph"` |
| 实体节点 | DocStore | 44 | `knowledge_graph_kwd="entity"` |
| 关系边 | DocStore | 78 | `knowledge_graph_kwd="relation"` |
| 社区报告 | DocStore | 5 | `knowledge_graph_kwd="community_report"` |
| Phase Marker | Redis | 1 | `phase_marker:{tenant}:{kb}` |
| 子图检查点 | Redis | 3 | `checkpoint:subgraph:{tenant}:{kb}:{hash}` |
| 消解检查点 | Redis | 2 | `checkpoint:resolution:{tenant}:{kb}` (Hash) |
| 社区检查点 | Redis | 5 | `checkpoint:community:{tenant}:{kb}` (Hash) |
| LLM 缓存 | Redis | ~20 | `llm_cache:{xxHash}` |
| Embed 缓存 | Redis | ~122 | `embed_cache:{xxHash}` (44实体+78边) |

### 性能分解

| 阶段 | 耗时（秒） | 占比 | 瓶颈 |
|-----|----------|------|------|
| 准备 | 2.3 | 0.9% | DocStore 查询 |
| 子图生成 | 85.4 | 34.8% | LLM 调用（3批次） |
| 子图合并 | 12.7 | 5.2% | 嵌入生成 + DocStore 插入 |
| 实体消解 | 68.2 | 27.8% | LLM 判断（2对） |
| 社区检测 | 77.0 | 31.3% | Leiden 算法 + LLM 报告生成（5个） |
| **总计** | **245.6** | **100%** | - |

### 后续查询示例

**1. 检索实体**
```python
query = {
    "kb_id": "kb_001",
    "knowledge_graph_kwd": ["entity"],
    "entity_kwd": "字节跳动"
}
result = ELASTICSEARCH.search(query)
# 返回实体详情和嵌入向量
```

**2. 向量检索相似实体**
```python
query_vec = embedding_model.encode("科技公司")
query = {
    "kb_id": "kb_001",
    "knowledge_graph_kwd": ["entity"],
    "vector": query_vec,
    "top_k": 10
}
results = ELASTICSEARCH.vector_search(query)
# 返回：["字节跳动", "腾讯", ...]
```

**3. 获取社区报告**
```python
query = {
    "kb_id": "kb_001",
    "knowledge_graph_kwd": ["community_report"],
    "entities_kwd": "字节跳动"
}
report = ELASTICSEARCH.search(query)
# 返回："字节跳动科技生态圈" 社区报告
```

---

## 断点续传示例

如果在阶段 3（实体消解）中途取消，重新运行时：

1. **加载 Phase Marker**：发现已完成到 `PHASE_RESOLUTION`
2. **跳过阶段 0-2**：直接从阶段 3 开始
3. **加载消解检查点**：已处理的实体对跳过
4. **继续未完成的工作**：从第 2 对实体开始
5. **完成后更新 Phase Marker**：标记为 `PHASE_COMMUNITY`
