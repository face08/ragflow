# RAGFlow GraphRAG 模块深度解析

本目录记录 RAGFlow 中 GraphRAG（知识图谱增强检索）模块的实现逻辑，覆盖从文档上传到知识图谱落库的完整链路。

## 文档索引

| 文档 | 内容 |
|---|---|
| [01-架构总览.md](./01-架构总览.md) | 模块定位、代码布局、核心概念、三种抽取器对比 |
| [02-任务调度链路.md](./02-任务调度链路.md) | 文档上传 → 任务入队 → task_executor 分发 → GraphRAG 入口 |
| [03-分块与批次构建.md](./03-分块与批次构建.md) | chunk 拉取、token 批次切分、增量判定 |
| [04-实体关系抽取.md](./04-实体关系抽取.md) | LLM 抽取、gleaning 复采、解析、节点/边合并 |
| [05-子图生成与合并.md](./05-子图生成与合并.md) | 子图落库、全局图合并、PageRank、分布式锁 |
| [06-实体消解.md](./06-实体消解.md) | 候选对生成、混合相似度、LLM 判定、连通分量合并 |
| [07-社区检测与报告.md](./07-社区检测与报告.md) | Leiden 聚类、社区报告生成、结构化输出 |
| [08-存储与检索.md](./08-存储与检索.md) | 图在文档库中的表示、embedding 预热、检索时的使用方式 |
| [09-容错与性能设计.md](./09-容错与性能设计.md) | 检查点、重试、超时、并发控制、缓存 |
| [10-完整流程时序图.md](./10-完整流程时序图.md) | 端到端时序、状态机、关键数据结构流转 |
| [11-架构设计.md](./11-架构设计.md) | 核心类图、数据流图、关键接口定义、依赖注入与扩展点 |
| [12-流程Demo-首次文档入库.md](./12-流程Demo-首次文档入库.md) | 首次文档入库完整流程，包含各阶段数据输出与存储位置 |
| [13-流程Demo-增量文档追加.md](./13-流程Demo-增量文档追加.md) | 增量文档追加流程，展示新旧实体合并与图更新逻辑 |
| [14-流程Demo-删除与重建.md](./14-流程Demo-删除与重建.md) | 删除与重建流程，包含单文档删除、知识库清空、知识库删除、仅清理缓存 |

## 快速理解

GraphRAG 在 RAGFlow 中是一个**独立于普通分块流程的后置任务**。普通解析任务把文档切成 chunk 并向量化入库；GraphRAG 任务则在这些 chunk 之上再跑一遍 LLM，抽取实体与关系，构造一张知识图谱，并把图谱本身也以 chunk 形式写回同一个文档库索引，从而让检索阶段可以同时命中「文本片段」和「图谱片段」。

一句话概括流程：

```
上传文档 → 解析分块 → (可选) GraphRAG 任务
   → 拉取 chunk → token 批次
   → LLM 抽实体/关系 → 单文档子图
   → 合并进知识库全局图 → PageRank
   → 实体消解（去重）
   → Leiden 社区检测 → 社区报告
   → 图谱写回文档库（带 embedding）
```

## 核心代码入口

- 任务入队：[task_service.py](file:///d:/work/rag/ragflow/api/db/services/task_service.py) 的 `queue_tasks`
- 任务分发：[task_executor.py](file:///d:/work/rag/ragflow/rag/svr/task_executor.py) 的 `do_handle_task`
- GraphRAG 主流程：[index.py](file:///d:/work/rag/ragflow/rag/graphrag/general/index.py) 的 `run_graphrag_for_kb`
- 抽取器基类：[extractor.py](file:///d:/work/rag/ragflow/rag/graphrag/general/extractor.py)
- 图工具与存储：[utils.py](file:///d:/work/rag/ragflow/rag/graphrag/utils.py)
