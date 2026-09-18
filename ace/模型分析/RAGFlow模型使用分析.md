# RAGFlow 模型使用分析

> 生成时间：2026-09-18  
> RAGFlow 版本：v0.27.2  
> 数据来源：conf/llm_factories.json, rag/llm/*.py

---

## 概述

RAGFlow 支持**多种类型的AI模型**，采用**插件化架构**，允许用户灵活选择本地部署或云端API。系统通过统一的抽象层（Base类）封装不同厂商的API差异，并通过 `llm_factories.json` 配置文件管理 **95+ 个模型提供商**。

**核心模型类型：**
- **对话模型（Chat/LLM）**：支持文本生成、多轮对话、推理
- **向量嵌入模型（Embedding）**：文本向量化，用于语义检索
- **重排序模型（Rerank）**：检索结果精排
- **多模态模型（Vision）**：图像理解（IMAGE2TEXT）
- **语音模型（Speech）**：语音转文字（SPEECH2TEXT）、文字转语音（TTS）
- **其他模型**：OCR、内容审核（Moderation）

---

## 1. 本地部署模型

### 1.1 Ollama

**作用：**
- 本地运行开源大语言模型的首选工具
- 支持 Llama、Qwen、Mistral、Gemma 等主流开源模型
- 通过 Docker 容器或本地进程运行

**技术架构：**
```python
# rag/llm/chat_model.py 和 embedding_model.py
class OllamaEmbed(Base):
    _FACTORY_NAME = "Ollama"
    # 使用 ollama Python SDK
    from ollama import Client
```

**配置信息：**
- **默认端口**：11434
- **API 兼容性**：OpenAI-compatible `/v1` endpoints
- **模型管理**：`ollama pull <model_name>` 下载模型

**向量维度：**
- 取决于具体模型，常见：
  - `nomic-embed-text`：768维
  - `mxbai-embed-large`：1024维
  - `bge-m3`：1024维

**功能特性：**
- ✅ 支持 Function Call（部分模型如 Qwen2.5 系列）
- ✅ 支持流式输出
- ✅ 支持多模态（qwen2-vl、llava 系列）
- ✅ 本地私有化部署，无数据泄露风险

**性能指标：**
- **推理速度**：取决于硬件（GPU/CPU）和模型大小
  - 7B 模型 + RTX 4090：~50-80 tokens/s
  - 13B 模型 + RTX 4090：~30-50 tokens/s
- **内存占耗**：7B模型约需 8-12GB VRAM（FP16）
- **优点**：完全本地化、成本可控、隐私安全
- **缺点**：需要硬件资源、模型性能通常低于商业闭源模型

---

### 1.2 LM Studio

**作用：**
- 图形化界面的本地模型管理工具
- 适合非技术用户快速部署本地模型
- 内置模型市场，一键下载

**配置信息：**
```python
_FACTORY_NAME = "LM-Studio"
# 默认端口：1234
# 兼容 OpenAI API
```

**功能特性：**
- ✅ 图形化模型管理
- ✅ 支持 Chat 和 Embedding 模型
- ✅ 支持 IMAGE2TEXT（多模态）
- ⚠️ Function Call 支持取决于加载的模型

**优缺点：**
- **优点**：用户友好、快速上手、跨平台
- **缺点**：性能优化不如 vLLM/Ollama、社区生态较小

---

### 1.3 LocalAI

**作用：**
- OpenAI API 的开源替代品
- 支持本地运行多种模型后端（llama.cpp、whisper.cpp 等）
- Docker 容器化部署

**配置信息：**
```python
_FACTORY_NAME = "LocalAI"
base_url: "http://localhost:8080/v1"
# 兼容 OpenAI SDK
```

**功能特性：**
- ✅ 多后端支持（llama.cpp、stablediffusion、whisper）
- ✅ 支持文本、语音、图像生成
- ✅ GPU 加速
- ⚠️ Function Call 支持取决于模型

**优缺点：**
- **优点**：功能全面、Docker 部署简单、社区活跃
- **缺点**：配置复杂度较高、文档分散

---

### 1.4 Xinference

**作用：**
- 阿里达摩院开源的模型推理框架
- 支持自动模型下载和管理
- 适合生产环境部署

**配置信息：**
```python
_FACTORY_NAME = "Xinference"
# 默认端口：9997
# 支持分布式部署
```

**功能特性：**
- ✅ 自动模型下载和版本管理
- ✅ 支持量化模型（int4/int8）
- ✅ 分布式推理
- ✅ 支持 Embedding 和 Rerank

**性能指标：**
- **吞吐量优化**：支持批处理、连续批处理
- **延迟**：取决于模型大小和硬件
- **优点**：生产级稳定性、易于扩展、监控完善
- **缺点**：学习曲线陡峭、需要一定运维能力

---

### 1.5 vLLM

**作用：**
- 高性能 LLM 推理引擎
- PagedAttention 技术大幅提升吞吐量
- 工业级推理性能

**配置信息：**
```python
_FACTORY_NAME = "VLLM"
base_url: "http://localhost:8000/v1"
# 兼容 OpenAI API
```

**性能指标：**
- **吞吐量**：相比原生 HuggingFace Transformers 提升 **10-24倍**
- **显存优化**：PagedAttention 减少显存碎片
- **并发能力**：支持高并发请求（100+ QPS）
- **优点**：极致性能、生产级稳定性
- **缺点**：仅支持推理（不支持训练）、配置参数较多

---

## 2. 云端模型集成

### 2.1 OpenAI

**作用：**
- 业界领先的商业大语言模型
- RAGFlow 的默认推荐选项

**支持模型（部分）：**
```json
{
  "gpt-5.5": "400K上下文，支持IMAGE2TEXT、Tools",
  "gpt-5.4": "400K上下文",
  "gpt-5.4-mini": "快速版本，400K",
  "gpt-5.4-nano": "超快版本，400K",
  "gpt-4.1": "1M上下文",
  "o3-pro": "推理模型，400K",
  "o3-mini": "推理模型，200K",
  "text-embedding-3-large": "Embedding，3072维",
  "text-embedding-3-small": "Embedding，1536维"
}
```

**向量维度：**
- `text-embedding-3-large`：**3072 维**（可通过参数降维到 256-3072）
- `text-embedding-3-small`：**1536 维**（可降维到 512-1536）
- `text-embedding-ada-002`：**1536 维**（传统模型）

**功能特性：**
- ✅ **Function Calling**：支持
- ✅ **Code Generation**：代码能力顶尖
- ✅ **Math Reasoning**：数学推理能力强

**性能指标：**
- **成本**：极低，约为 GPT-4 的 1/10
- **推理速度**：快速响应
- **优点**：开源可商用、性价比极高、代码和数学能力突出
- **缺点**：多模态能力较弱、生态相对较小

---

### 2.6 Google Gemini

**作用：**
- Google 旗舰大模型
- 原生多模态架构

**支持模型：**
```json
{
  "gemini-3.5-ultra": "2M上下文",
  "gemini-3.5-pro": "2M上下文，性价比",
  "gemini-3.5-flash": "1M，极速响应",
  "gemini-3.5-nano": "边缘设备",
  "text-embedding-005": "Embedding，768维"
}
```

**向量维度：**
- `text-embedding-005`：**768 维**
- `text-embedding-004`：**768 维**

**功能特性：**
- ✅ **Function Calling**：支持
- ✅ **Native Multimodal**：原生支持文本/图像/音频/视频
- ✅ **Ultra Long Context**：2M token 窗口
- ✅ **Grounding**：Google 搜索增强

**性能指标：**
- **上下文长度**：业界领先（2M）
- **多模态能力**：强于 GPT-4V
- **优点**：超长上下文、原生多模态、Google 生态整合
- **缺点**：国内访问受限、API 稳定性略低于 OpenAI

---

### 2.7 AWS Bedrock

**作用：**
- AWS 托管的多模型平台
- 企业级合规和安全

**支持模型（通过 Bedrock 访问）：**
- Anthropic Claude 系列
- Amazon Titan 系列
- Meta Llama 系列
- Cohere Command 系列
- Stability AI 系列

**功能特性：**
- ✅ **Multi-provider**：一个接口访问多家模型
- ✅ **Enterprise Grade**：合规性、数据驻留、审计日志
- ✅ **Guardrails**：内容过滤和安全策略
- ✅ **Fine-tuning**：支持自定义微调

**优缺点：**
- **优点**：企业级、合规性好、AWS 生态整合
- **缺点**：成本较高、区域限制、需 AWS 账号

---

### 2.8 其他国际主流厂商

#### Cohere
- **专长**：企业级 NLP、Embedding 和 Rerank
- **模型**：`command-r-plus`（128K）、`command-r`（128K）
- **Embedding**：`embed-v4`（1024维）、`embed-multilingual-v3.0`（1024维）
- **Rerank**：`rerank-v3.5`、`rerank-multilingual-v3.0`
- **特点**：多语言支持优秀、企业部署友好

#### Moonshot（月之暗面）
- **模型**：`moonshot-v1-128k`、`moonshot-v1-32k`、`moonshot-v1-8k`
- **特点**：超长上下文（128K）、中文优化
- **成本**：国内中等价位

#### Minimax
- **模型**：`abab6.5`、`abab6.5-chat`
- **特点**：语音合成（TTS）能力强
- **应用场景**：智能客服、语音交互

---

## 3. 专用 Embedding 模型

### 3.1 内置模型（Builtin）

**作用：**
- RAGFlow 内置的开源 Embedding 模型
- 本地运行，无需外部 API

**支持模型：**
```python
# rag/llm/embedding_model.py
{
  "BAAI/bge-large-zh-v1.5": "1024维，中文优化",
  "BAAI/bge-base-en-v1.5": "768维，英文",
  "BAAI/bge-small-en-v1.5": "384维，快速",
  "BAAI/bge-m3": "1024维，多语言",
  "Qwen/Qwen3-Embedding-0.6B": "1536维，高质量"
}
```

**向量维度：**
- BGE Large：**1024 维**
- BGE Base：**768 维**
- BGE Small：**384 维**
- BGE-M3：**1024 维**
- Qwen3-Embedding：**1536 维**

**性能指标：**
- **推理速度**：取决于硬件
  - GPU (RTX 4090)：~1000 文档/秒
  - CPU：~50-100 文档/秒
- **质量**：接近商业模型（在中文场景）
- **优点**：免费、私有化、中文优化好
- **缺点**：需要本地资源、英文质量略低于 OpenAI

---

### 3.2 Jina AI

**作用：**
- 专注于 Embedding 和 Rerank 的创业公司
- 多模态 Embedding 支持

**支持模型：**
```json
{
  "jina-embeddings-v3": "1024维，最新",
  "jina-clip-v2": "多模态（文本+图像）",
  "jina-reranker-v2-base-multilingual": "Rerank"
}
```

**向量维度：**
- `jina-embeddings-v3`：**1024 维**
- `jina-clip-v2`：**768 维**（图文共享空间）

**功能特性：**
- ✅ **Multimodal**：图文联合 Embedding
- ✅ **Long Context**：支持 8K token
- ✅ **Multilingual**：多语言优化

**优缺点：**
- **优点**：多模态能力、API 简单、价格友好
- **缺点**：生态较小、知名度不如 OpenAI

---

### 3.3 Voyage AI

**作用：**
- Anthropic 官方推荐的 Embedding 提供商
- 专为 RAG 场景优化

**支持模型：**
```json
{
  "voyage-3": "1024维，通用",
  "voyage-3-lite": "512维，快速",
  "voyage-code-3": "1024维，代码专用",
  "voyage-finance-2": "1024维，金融领域"
}
```

**向量维度：**
- 标准版：**1024 维**
- Lite 版：**512 维**

**功能特性：**
- ✅ **Domain-Specific**：领域优化（代码、金融、法律）
- ✅ **RAG Optimized**：专为检索增强生成优化
- ✅ **High Quality**：质量接近 OpenAI text-embedding-3

**优缺点：**
- **优点**：RAG 场景表现优秀、领域定制化
- **缺点**：价格略高、生态较新

---

## 4. Rerank 模型

### 4.1 Cohere Rerank

**作用：**
- 业界领先的重排序模型
- 显著提升检索精度

**支持模型：**
```python
# rag/llm/rerank_model.py
{
  "rerank-v3.5": "多语言，最新",
  "rerank-english-v3.0": "英文专用",
  "rerank-multilingual-v3.0": "多语言"
}
```

**技术细节：**
```python
class CohereRerank(Base):
    def similarity(self, query: str, texts: List) -> Tuple[np.ndarray, int]:
        # 返回 [0, 1] 归一化分数
        # 使用交叉注意力机制（query-document interaction）
```

**性能指标：**
- **精度提升**：相比向量检索单独使用，提升 **20-40% NDCG@10**
- **延迟**：~100-300ms（100 文档）
- **优点**：精度高、多语言支持
- **缺点**：需要额外 API 调用、增加延迟

---

### 4.2 Jina Rerank

**作用：**
- 开源友好的重排序方案

**支持模型：**
```json
{
  "jina-reranker-v2-base-multilingual": "多语言",
  "jina-reranker-v2-base-en": "英文专用"
}
```

**优缺点：**
- **优点**：价格低、支持自部署
- **缺点**：精度略低于 Cohere

---

### 4.3 国内 Rerank 方案

#### 通义千问 Rerank
- **模型**：`gte-rerank-v2`、`qwen3-rerank`
- **特点**：中文优化、价格友好
- **向量维度**：N/A（Rerank 不输出向量）

#### BAAI BGE Rerank（本地）
- **模型**：`BAAI/bge-reranker-large`、`BAAI/bge-reranker-base`
- **特点**：完全本地化、免费
- **性能**：中文场景接近商业模型

---

## 5. 多模态模型（Vision）

### 5.1 OpenAI GPT-4 Vision

**作用：**
- 图像理解和文档解析
- OCR 和视觉问答

**支持模型：**
- `gpt-5.5`（支持 IMAGE2TEXT）
- `gpt-4-turbo`（支持 Vision）
- `gpt-4o`（优化的多模态）

**功能特性：**
- ✅ **OCR**：文档识别
- ✅ **Image Understanding**：场景理解、对象检测
- ✅ **Chart Analysis**：图表解析
- ✅ **Function Calling**：结合视觉输入调用工具

**应用场景：**
- 扫描文档解析
- 图片内容理解
- 图表数据提取

---

### 5.2 通义千问 VL（Qwen-VL）

**作用：**
- 国内领先的多模态模型
- 中文图像理解优化

**支持模型：**
```json
{
  "qwen-vl-max": "最强视觉模型",
  "qwen-vl-plus": "平衡性价比"
}
```

**功能特性：**
- ✅ **Chinese OCR**：中文 OCR 优化
- ✅ **Grounding**：视觉定位
- ✅ **Multi-image**：多图理解

**优缺点：**
- **优点**：中文 OCR 准确、价格友好、国内访问快
- **缺点**：国际场景表现略逊于 GPT-4V

---

## 6. 语音模型

### 6.1 OpenAI Whisper（SPEECH2TEXT）

**作用：**
- 语音转文字
- 多语言支持

**支持模型：**
- `whisper-1`：通用版本

**功能特性：**
- ✅ **Multi-language**：支持 99 种语言
- ✅ **High Accuracy**：准确率高
- ✅ **Timestamp**：支持时间戳输出

**性能指标：**
- **延迟**：实时转写（流式）或批量处理
- **准确率**：中英文均优秀
- **优点**：多语言、开源可部署
- **缺点**：API 版本成本较高

---

### 6.2 通义千问 ASR

**作用：**
- 中文语音识别优化

**支持模型：**
- `qwen3-asr`：最新语音识别模型

**功能特性：**
- ✅ **Chinese Optimized**：中文方言支持
- ✅ **Real-time**：实时转写
- ✅ **Punctuation**：自动标点

**优缺点：**
- **优点**：中文准确率高、价格低
- **缺点**：英文场景略逊于 Whisper

---

### 6.3 TTS（文字转语音）

#### OpenAI TTS
- **模型**：`tts-1`、`tts-1-hd`
- **声音**：支持多种音色（alloy、echo、nova 等）
- **特点**：自然度高、延迟低

#### 通义千问 TTS
- **模型**：`sambert-zhichu-v1`、`sambert-zhitian-v1`
- **声音**：中文音色丰富
- **特点**：中文情感表现力强

---

## 7. 性能对比与选型指南

### 7.1 Chat/LLM 模型对比矩阵

| 模型 | 上下文 | Function Call | 成本/1M tokens | 推理速度 | 适用场景 |
|------|--------|---------------|---------------|---------|---------|
| **OpenAI GPT-5.5** | 400K | ✅ | $15-30 | 快 | 通用、最强性能 |
| **Claude Opus 5** | 200K | ✅ | $15 | 中 | 长文本、企业应用 |
| **Gemini 3.5 Pro** | 2M | ✅ | $10 | 快 | 超长上下文 |
| **通义千问 Qwen3-Max** | 128K | ✅ | ¥0.04 | 快 | 国内、性价比 |
| **智谱 GLM-5** | 128K | ✅ | ¥0.05 | 快 | 中文、代码 |
| **DeepSeek Chat** | 64K | ✅ | ¥0.001 | 快 | 极致性价比 |
| **Ollama (本地)** | 取决于模型 | ⚠️ | 免费 | 慢 | 私有化、开发测试 |

---

### 7.2 Embedding 模型对比

| 模型 | 向量维度 | 成本/1M tokens | 质量（英文） | 质量（中文） | 推荐场景 |
|------|---------|---------------|------------|------------|---------|
| **OpenAI text-embedding-3-large** | 3072 | $0.13 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 英文为主、高精度 |
| **OpenAI text-embedding-3-small** | 1536 | $0.02 | ⭐⭐⭐⭐ | ⭐⭐⭐ | 性价比平衡 |
| **通义 text-embedding-v4** | 可调 | ¥0.0007 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 中文优先、低成本 |
| **智谱 embedding-4** | 可调 | ¥0.0005 | ⭐⭐⭐ | ⭐⭐⭐⭐ | 中文、极低成本 |
| **BGE-M3 (本地)** | 1024 | 免费 | ⭐⭐⭐ | ⭐⭐⭐⭐ | 私有化、多语言 |
| **Jina Embeddings v3** | 1024 | $0.02 | ⭐⭐⭐⭐ | ⭐⭐⭐ | 长文本、多模态 |

---

### 7.3 Rerank 模型对比

| 模型 | 精度提升 | 延迟 | 成本 | 多语言 | 推荐场景 |
|------|---------|------|------|--------|---------|
| **Cohere Rerank v3.5** | ⭐⭐⭐⭐⭐ | 中 | 中 | ✅ | 高精度要求 |
| **通义 gte-rerank-v2** | ⭐⭐⭐⭐ | 低 | 低 | ✅ | 中文、性价比 |
| **Jina Rerank v2** | ⭐⭐⭐⭐ | 中 | 低 | ✅ | 开源友好 |
| **BGE Reranker (本地)** | ⭐⭐⭐ | 低 | 免费 | ⚠️ | 私有化、中文 |

---

## 8. 使用建议与最佳实践

### 8.1 场景化选型建议

#### 场景 1：企业级 RAG 系统（高精度、合规）
**推荐配置：**
- **Chat**：OpenAI GPT-5.5 或 Claude Opus 5
- **Embedding**：OpenAI text-embedding-3-large
- **Rerank**：Cohere Rerank v3.5
- **理由**：性能最优、API 稳定、企业级 SLA

#### 场景 2：国内生产环境（合规、中文优化）
**推荐配置：**
- **Chat**：通义千问 Qwen3-Max 或 智谱 GLM-5
- **Embedding**：通义 text-embedding-v4
- **Rerank**：通义 gte-rerank-v2
- **理由**：国内合规、中文优化、成本低、访问快

#### 场景 3：私有化部署（数据安全、成本敏感）
**推荐配置：**
- **Chat**：Ollama (Qwen2.5-14B / Llama-3.1-8B)
- **Embedding**：BGE-M3 或 Qwen3-Embedding (本地)
- **Rerank**：BGE Reranker (本地)
- **理由**：完全私有化、无数据泄露、长期成本低

#### 场景 4：极致性价比（初创、实验）
**推荐配置：**
- **Chat**：DeepSeek Chat
- **Embedding**：智谱 embedding-4
- **Rerank**：通义 gte-rerank-v2
- **理由**：成本极低（约为 OpenAI 的 1/10-1/20）

#### 场景 5：多模态应用（文档解析、图片理解）
**推荐配置：**
- **Vision**：OpenAI GPT-4o 或 通义 Qwen-VL-Max
- **OCR**：通义 Qwen-VL（中文优势）
- **理由**：多模态能力强、文档理解准确

---

### 8.2 技术实现要点

#### LiteLLM 集成模式
```python
# RAGFlow 通过 LiteLLM 统一不同厂商 API
from litellm import completion

response = completion(
    model="tongyi-qianwen/qwen3-max",  # 前缀路由
    messages=[{"role": "user", "content": "Hello"}],
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)
```

#### 向量维度选择原则
- **高精度场景**：选择 1024+ 维（OpenAI 3072维、通义 v4）
- **平衡场景**：512-1024 维（Jina、BGE）
- **快速检索**：384-512 维（BGE-small、Voyage-lite）
- **原则**：维度越高，语义表达越精确，但存储和计算成本越高

#### Function Calling 支持判断
```python
# 查看 conf/llm_factories.json 中的 is_tools 字段
{
  "llm_name": "gpt-5.5",
  "is_tools": true  # ✅ 支持 Function Calling
}
```

#### Claude 采样参数特殊处理
```python
# RAGFlow 自动处理 Claude 的采样限制
# Opus 4.7+/Sonnet 5：不传任何采样参数
# Claude 4.1+：只传 temperature，不传 top_p
# 代码位置：rag/llm/chat_model.py::_apply_claude_sampling_policy
```

---

### 8.3 成本优化策略

#### 策略 1：混合部署
- **高频简单查询**：使用本地 Ollama 或 DeepSeek（低成本）
- **复杂推理任务**：使用 GPT-5 或 Claude（按需调用）
- **预期节省**：60-80% API 成本

#### 策略 2：Embedding 缓存
- 对相同文档的 Embedding 结果进行缓存
- 避免重复向量化
- **实现**：使用 Redis 缓存向量结果

#### 策略 3：Rerank 分层
- **第一层**：向量检索（快速召回 Top 100）
- **第二层**：Rerank 精排（只对 Top 10-20 重排）
- **效果**：降低 Rerank API 调用次数 80%

#### 策略 4：批处理
```python
# OpenAI Embedding 批处理限制
# 每批最多 16 个文本，单文本最多 8191 tokens
class OpenAIEmbed(Base):
    def encode(self, texts: list):
        return self._batched_encode(
            texts, self._call, 
            batch_size=16,      # 批量处理
            truncate_to=8191    # token 截断
        )
```

---

### 8.4 性能优化建议

#### 本地部署优化
1. **硬件选择**
   - **推理**：NVIDIA RTX 4090 / A100（24GB+ VRAM）
   - **Embedding**：支持 INT8 量化的 GPU
   - **内存**：模型大小的 1.5-2 倍

2. **量化技术**
   - **INT8**：性能损失 <2%，显存减半
   - **INT4**：性能损失 5-10%，显存减至 1/4
   - **工具**：使用 vLLM 的自动量化

3. **并发优化**
   - **vLLM**：PagedAttention，支持 100+ QPS
   - **Ollama**：单实例并发有限，建议负载均衡
   - **Xinference**：原生支持分布式

#### API 调用优化
1. **连接池**：复用 HTTP 连接
2. **超时设置**：合理设置超时（30-60s）
3. **重试机制**：指数退避重试
4. **限流保护**：客户端限流，避免触发 Rate Limit

---

## 9. 完整支持列表

### 9.1 全部 95+ 提供商概览

**本地部署（7家）：**
Ollama, LM-Studio, LocalAI, Xinference, VLLM, HuggingFace, Together AI

**国际主流（25家）：**
OpenAI, Anthropic, Google (Gemini/VertexAI), AWS Bedrock, Azure, Cohere, Mistral, xAI, Perplexity, Groq, Fireworks AI, Anyscale, DeepInfra, Replicate, AI21, Writer, NLP Cloud, Aleph Alpha, Petals, Voyage AI, Jina AI, Baseten, vLLM, Cloudflare

**国内厂商（15家）：**
Tongyi-Qianwen (阿里), ZHIPU-AI (智谱), DeepSeek, Moonshot (月之暗面), Minimax, Baichuan (百川), Qwen (通义), Sensenova (商汤), Yi (零一万物), Baidu (文心一言), Hunyuan (腾讯混元), Spark (讯飞星火), ChatGLM, MiniMax, Stepfun (阶跃星辰)

**云服务平台（10家）：**
AWS (Bedrock/Sagemaker), Azure, Google Cloud (VertexAI), Alibaba Cloud, Tencent Cloud, Huawei Cloud, Oracle Cloud, IBM watsonx, Databricks, Cloudflare Workers AI

**专业领域（8家）：**
- **金融**：BloombergGPT (通过 Bedrock)
- **医疗**：Med-PaLM (通过 VertexAI)
- **代码**：Codestral, CodeLlama, StarCoder
- **嵌入**：Voyage, Jina, Cohere Embed
- **语音**：OpenAI Whisper, ElevenLabs, PlayHT

**开源托管（30家）：**
HuggingFace, Together AI, Replicate, Baseten, Modal, RunPod, Lambda Labs, Paperspace, Banana, Inferless 等

---

### 9.2 按功能分类支持矩阵

#### Function Calling 支持（45+ 模型）
- OpenAI GPT-4/5 系列 ✅
- Anthropic Claude 4/5 系列 ✅
- Google Gemini 系列 ✅
- 通义千问 Qwen3 系列 ✅
- 智谱 GLM-4/5 系列 ✅
- DeepSeek 系列 ✅
- Mistral Large ✅
- Cohere Command R+ ✅

#### Vision (IMAGE2TEXT) 支持（20+ 模型）
- GPT-4o/4-turbo/5.5
- Claude Opus/Sonnet (部分)
- Gemini Pro/Ultra/Flash
- Qwen-VL-Max/Plus
- GLM-4V
- Llava (本地)

#### Long Context (100K+) 支持
- **10M tokens**：Qwen-Long
- **2M tokens**：Gemini 3.5 Pro/Ultra
- **1M tokens**：GPT-4.1
- **400K tokens**：GPT-5 系列
- **200K tokens**：Claude 系列
- **128K tokens**：Qwen3, GLM-5, Moonshot, Command R+

---

## 10. 总结与未来趋势

### 10.1 核心要点

1. **模型生态丰富**：RAGFlow 支持 95+ 提供商，覆盖全球主流模型
2. **多模型类型**：Chat、Embedding、Rerank、Vision、Speech 全覆盖
3. **灵活部署**：支持本地部署（Ollama/vLLM）和云端 API 混合使用
4. **成本可控**：从免费本地到商业 API，提供多层次选择
5. **国内优化**：深度集成通义千问、智谱等国内厂商，合规友好

### 10.2 技术趋势

1. **超长上下文**：从 128K → 1M → 10M tokens
2. **多模态融合**：文本+图像+音频+视频统一模型
3. **本地化加速**：开源模型（Llama 3.1、Qwen2.5）逼近商业模型
4. **成本下降**：API 价格持续下降（2024年降幅 50-70%）
5. **Function Calling 普及**：从高端模型下沉到中低端模型

### 10.3 选型决策树

```
是否需要数据私有化？
├─ 是 → 本地部署（Ollama/vLLM + BGE）
└─ 否 → 
    ├─ 主要用户在国内？
    │   ├─ 是 → 通义千问/智谱 GLM
    │   └─ 否 → OpenAI/Claude/Gemini
    └─ 成本敏感？
        ├─ 是 → DeepSeek/智谱
        └─ 否 → OpenAI GPT-5/Claude Opus
```

---

## 附录

### A. 配置文件位置
- **模型定义**：`conf/llm_factories.json`
- **Chat 实现**：`rag/llm/chat_model.py`
- **Embedding 实现**：`rag/llm/embedding_model.py`
- **Rerank 实现**：`rag/llm/rerank_model.py`
- **Provider 枚举**：`rag/llm/__init__.py`

### B. 关键技术术语
- **LiteLLM**：统一多厂商 API 的抽象层
- **Function Calling**：模型主动调用外部工具的能力
- **Rerank**：对召回结果进行二次排序以提升精度
- **PagedAttention**：vLLM 的显存优化技术
- **Embedding**：将文本转换为向量表示
- **Context Window**：模型能处理的最大 token 数

### C. 参考资源
- RAGFlow 官方文档：https://ragflow.io/docs
- LiteLLM 文档：https://docs.litellm.ai
- 模型性能对比：https://artificialanalysis.ai
- 向量数据库选型：参见 `ace/存储分析/RAGFlow存储使用分析.md`

---

**文档版本**：v1.0  
**最后更新**：2026-09-18  
**维护者**：RAGFlow 分析团队

---

## 附录 D：OpenAI 完整功能特性

### 2.1 OpenAI（补充完整版）

**作用：**
- 业界领先的商业大语言模型
- RAGFlow 的默认推荐选项

**支持模型（部分）：**
```json
{
  "gpt-5.5": "400K上下文，支持IMAGE2TEXT、Tools",
  "gpt-5.4": "400K上下文",
  "gpt-5.4-mini": "快速版本，400K",
  "gpt-5.4-nano": "超快版本，400K",
  "gpt-4.1": "1M上下文",
  "o3-pro": "推理模型，400K",
  "o3-mini": "推理模型，200K",
  "text-embedding-3-large": "Embedding，3072维",
  "text-embedding-3-small": "Embedding，1536维"
}
```

**向量维度：**
- `text-embedding-3-large`：**3072 维**（可通过参数降维到 256-3072）
- `text-embedding-3-small`：**1536 维**（可降维到 512-1536）
- `text-embedding-ada-002`：**1536 维**（传统模型）

**功能特性：**
- ✅ **Function Calling**：支持工具调用（`is_tools: true`）
- ✅ **Vision**：gpt-4 系列支持图像理解
- ✅ **Streaming**：流式输出
- ✅ **JSON Mode**：结构化输出（`response_format`）
- ✅ **Reasoning**：o3 系列支持推理链
- ✅ **Moderation**：内容审核 API

**性能指标：**
- **API 延迟**：50-500ms（取决于模型和负载）
- **速率限制**：按账户级别（TPM/RPM）
- **可靠性**：99.9% SLA
- **成本**：
  - GPT-5 系列：$10-30/1M tokens
  - GPT-4 系列：$2.5-10/1M tokens
  - Embedding：$0.02-0.13/1M tokens

**优缺点：**
- **优点**：性能顶尖、功能完善、API 稳定、文档详尽
- **缺点**：成本较高、需要网络连接、数据隐私顾虑、部分地区访问受限

---

## 附录 E：Anthropic Claude 完整信息

### 2.2 Anthropic Claude（补充完整版）

**作用：**
- OpenAI 的主要竞争对手
- 长上下文处理能力突出
- 注重安全和可解释性

**支持模型：**
```json
{
  "claude-opus-5": "200K上下文，最强推理",
  "claude-opus-4-8": "200K上下文",
  "claude-sonnet-4-6": "200K上下文，平衡性价比",
  "claude-haiku-4.5": "200K上下文，快速响应"
}
```

**向量维度：**
- Anthropic 不直接提供 Embedding API
- 通过 Voyage AI 合作提供（见 3.3 节）

**功能特性：**
- ✅ **Function Calling**：支持工具调用
- ✅ **Vision**：支持图像理解（部分模型）
- ✅ **Long Context**：原生 200K token 窗口
- ⚠️ **特殊限制**：
  - Claude Opus 4.7+ / Sonnet 5 / Fable / Mythos：**不接受任何采样参数**（temperature/top_p/top_k）
  - Claude 4.1+：**不能同时设置 temperature 和 top_p**

**采样策略处理：**
```python
# rag/llm/chat_model.py
def _apply_claude_sampling_policy(model_name_lower: str, *targets: dict):
    # Opus 4.7+, Sonnet 5: 删除所有采样参数
    # Claude 4.1+: 删除 top_p（保留 temperature）
```

**性能指标：**
- **API 延迟**：类似 OpenAI
- **长文本处理**：优于 GPT-4（200K 原生窗口）
- **成本**：与 GPT-4 相当

**优缺点：**
- **优点**：长上下文、安全对齐好、适合企业应用
- **缺点**：无原生 Embedding、国内访问困难

**支持模型：**
```json
{
  "claude-opus-5": "200K上下文，最强推理",
  "claude-opus-4-8": "200K上下文",
  "claude-sonnet-4-6": "200K上下文，平衡性价比",
  "claude-haiku-4.5": "200K上下文，快速响应"
}
```

**向量维度：**
- Anthropic 不直接提供 Embedding API
- 通过 Voyage AI 合作提供（见 2.9 节）

**功能特性：**
- ✅ **Function Calling**：支持工具调用
- ✅ **Vision**：支持图像理解（部分模型）
- ✅ **Long Context**：原生 200K token 窗口
- ⚠️ **特殊限制**：
  - Claude Opus 4.7+ / Sonnet 5 / Fable / Mythos：**不接受任何采样参数**（temperature/top_p/top_k）
  - Claude 4.1+：**不能同时设置 temperature 和 top_p**

**采样策略处理：**
```python
# rag/llm/chat_model.py
def _apply_claude_sampling_policy(model_name_lower: str, *targets: dict):
    # Opus 4.7+, Sonnet 5: 删除所有采样参数
    # Claude 4.1+: 删除 top_p（保留 temperature）
```

**性能指标：**
- **API 延迟**：类似 OpenAI
- **长文本处理**：优于 GPT-4（200K 原生窗口）
- **成本**：与 GPT-4 相当

**优缺点：**
- **优点**：长上下文、安全对齐好、适合企业应用
- **缺点**：无原生 Embedding、国内访问困难

---

### 2.3 国内厂商：通义千问（Tongyi-Qianwen / Dashscope）

**作用：**
- 阿里云旗下大模型
- 国内访问稳定，合规性好

**支持模型：**
```json
{
  "qwen3-max": "128K，最强性能",
  "qwen3-plus": "128K，性价比",
  "qwen-long": "10M上下文（1000万token）",
  "qwen-turbo": "1M，快速推理",
  "text-embedding-v3": "8K，1024维",
  "text-embedding-v4": "8K，向量维度可调",
  "gte-rerank-v2": "重排序模型",
  "qwen-vl-max": "多模态"
}
```

**向量维度：**
- `text-embedding-v3`：**1024 维**
- `text-embedding-v4`：**可配置维度**（支持自定义）

**功能特性：**
- ✅ **Function Calling**：支持工具调用
- ✅ **Vision**：qwen-vl 系列
- ✅ **Long Context**：qwen-long 支持 10M token
- ✅ **TTS**：sambert 系列语音合成
- ✅ **ASR**：qwen3-asr 语音识别
- ✅ **Rerank**：gte-rerank-v2 和 qwen3-rerank

**特殊处理：**
```python
# DashScope 国内/国际端点自动切换
# 国内：https://dashscope.aliyuncs.com/api/v1
# 国际：https://dashscope-intl.aliyuncs.com/api/v1
```

**性能指标：**
- **API 延迟**：国内 50-200ms
- **成本**：相比 OpenAI 便宜 50-70%
- **优点**：国内合规、中文优化、价格友好
- **缺点**：国际版性能略低于国内版

---

### 2.4 国内厂商：智谱 AI（ZHIPU-AI）

**作用：**
- 清华系大模型，GLM 系列
- 企业级应用广泛

**支持模型：**
```json
{
  "glm-5.2": "128K，最新旗舰",
  "glm-5.1": "128K",
  "glm-5-turbo": "128K，快速版",
  "glm-4.7": "128K",
  "embedding-4": "Embedding，维度可调"
}
```

**向量维度：**
- `embedding-4`：**1024/2048/3072 维可选**
- `embedding-3`：**2048 维**
- `embedding-2`：**1024 维**

**功能特性：**
- ✅ **Function Calling**：GLM-4 及以上支持
- ✅ **Vision**：GLM-4V 系列
- ✅ **Code Interpreter**：支持代码执行
- ✅ **Web Search**：联网搜索能力

**性能指标：**
- **推理速度**：与通义千问相当
- **成本**：GLM-5 约 ¥0.05/1K tokens
- **优点**：中文能力强、代码生成优秀
- **缺点**：国际化不足

---

### 2.5 国内厂商：DeepSeek

**作用：**
- 高性价比开源模型
- 数学和代码能力突出

**支持模型：**
```json
{
  "deepseek-chat": "64K，通用对话",
  "deepseek-coder": "代码专用",
  "deepseek-reasoner": "推理模型"
}
```

**功能特性：**
- ✅ **Function Calling**：支持