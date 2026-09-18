# RAGFlow Memory 模块深度解析

本目录记录 RAGFlow 中 Memory（记忆）模块的实现逻辑，覆盖从 Memory 创建、消息写入、异步提取、检索查询到遗忘淘汰的完整链路。

## 文档索引

| 文档 | 内容 |
|---|---|
| [01-架构总览.md](./01-架构总览.md) | 双运行时架构、数据模型、记忆类型、权限模型、索引策略 |
| [02-创建与配置流程.md](./02-创建与配置流程.md) | CreateMemory / UpdateMemory / DeleteMemory 的完整流程 |
| [03-消息写入流程.md](./03-消息写入流程.md) | Redis INCR 消息 ID、原始消息构建、任务入队、NATS 唤醒 |
| [04-异步提取流程.md](./04-异步提取流程.md) | 任务声明、租约机制、LLM 提取、重试策略、协调器 |
| [05-检索流程.md](./05-检索流程.md) | 语义检索、混合搜索、消息列表、类型过滤、时间范围 |
| [06-遗忘与淘汰机制.md](./06-遗忘与淘汰机制.md) | 软删除 forget_at、FIFO 自动淘汰、Redis 缓存、配额管理 |

## 快速理解

Memory 在 RAGFlow 中是一个**会话级的持久化记忆系统**，支持多种记忆类型（原始、语义、情节、过程）并通过异步 LLM 提取机制从用户原始消息中抽取结构化知识。系统采用双运行时架构（Go + Python），通过数据库状态机和 NATS 消息队列实现分布式任务协调。

一句话概括流程：

```
创建 Memory → 写入用户消息（Redis INCR ID）
   → 入队异步提取任务 → worker 声明租约
   → LLM 提取结构化记忆 → 持久化提取结果
   → 检索查询（语义 + 文本混合搜索）
   → 配额溢出触发 FIFO 淘汰
   → 用户主动遗忘（forget_at 软删除）
```

## 核心特性

- **双运行时架构**：Go 处理 HTTP 和任务协调，Python 处理嵌入和 LLM 调用，功能锁步一致
- **记忆类型**：原始消息、语义记忆、情节记忆、过程记忆，使用位标志组合
- **分布式任务**：基于数据库租约的 SELECT FOR UPDATE 声明机制，2 分钟 TTL，30 秒心跳续约
- **混合检索**：BM25 文本相似度 + cosine 向量相似度，weighted_sum 融合（默认各 0.5 权重）
- **配额管理**：5MB 默认限制，写入前 FIFO 淘汰，Redis 原子计数缓存
- **遗忘机制**：软删除（forget_at）保留数据供审计，物理淘汰释放配额

## 核心代码入口

- Memory CRUD：[memory.go](file:///d:/work/rag/ragflow/internal/service/memory.go) 的 `CreateMemory` / `UpdateMemory` / `DeleteMemory`
- 消息写入：[memory_message_service.go](file:///d:/work/rag/ragflow/internal/service/memory_message_service.go) 的 `QueueSaveToMemoryTask`
- 异步提取：[memory_extractor.go](file:///d:/work/rag/ragflow/internal/service/memory_extractor.go) 的 `HandleSaveToMemoryTask`
- 检索查询：[memory.go](file:///d:/work/rag/ragflow/internal/service/memory.go) 的 `queryMessage` / `listMemoryMessages`
- 淘汰逻辑：[memory_message_service.py](file:///d:/work/rag/ragflow/api/db/joint_services/memory_message_service.py) 的 `embed_and_save`
- FIFO 策略：[messages.py](file:///d:/work/rag/ragflow/memory/services/messages.py) 的 `pick_messages_to_delete_by_fifo`

## 技术亮点

1. **Redis INCR 消息 ID**：避免分布式 ID 冲突，单 Memory 内严格递增
2. **租约续约机制**：worker 每 30 秒续约，超时自动释放供其他节点接管
3. **指数退避重试**：5s → 10s → 20s → 40s → 80s → 5min × 5，最多 10 次
4. **协调器兜底**：每 5 分钟扫描超时和卡住任务，确保最终一致性
5. **写前淘汰**：先清理空间再写入，保证写入后总大小不超配额
6. **两阶段淘汰**：优先清理已遗忘消息，不足时按 valid_at 升序清理活跃消息

## 数据流示意

```
┌──────────────┐
│ 用户发送消息  │
└──────┬───────┘
       ↓
┌─────────────────────┐
│ Redis INCR 生成 ID   │
└──────┬──────────────┘
       ↓
┌─────────────────────┐
│ 构建原始消息文档     │
└──────┬──────────────┘
       ↓
┌─────────────────────┐
│ 插入 MemoryTask      │
└──────┬──────────────┘
       ↓
┌─────────────────────┐
│ NATS 发送唤醒通知    │
└──────┬──────────────┘
       ↓
┌─────────────────────┐
│ Worker 声明租约      │
└──────┬──────────────┘
       ↓
┌─────────────────────┐
│ LLM 提取结构化记忆   │
└──────┬──────────────┘
       ↓
┌─────────────────────┐
│ 持久化提取结果       │
└──────┬──────────────┘
       ↓
┌─────────────────────┐
│ 标记任务完成         │
└─────────────────────┘
```

---

文档版本：1.0  
最后更新：2026-09-18  
基于代码版本：RAGFlow 主分支
