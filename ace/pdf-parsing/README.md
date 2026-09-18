# RAGFlow PDF 解析模块详解

本目录记录 RAGFlow 中 PDF 文档解析的完整流程，从 OCR 文字识别到最终 Chunk 生成与存储的全链路。

## 文档索引

| 文档 | 内容 |
|---|---|
| [01-完整解析流程.md](./01-完整解析流程.md) | 完整的 7 阶段 PDF 解析流程，每个阶段的输入输出数据结构、最终 Chunk 格式、存储索引结构和检索高亮流程 |

## 快速理解

PDF 解析是 RAGFlow 文档解析中最复杂的部分，需要经过 **7 个主要阶段**：

```
PDF 文件
  ↓
[阶段 1] OCR 文字识别 → 提取文本和坐标
  ↓
[阶段 2] 布局分析 → 识别标题、正文、表格、图片等
  ↓
[阶段 3] 表格结构识别 → 解析表格的行列结构
  ↓
[阶段 4] 文本合并 → 将碎片化的文本框合并成段落
  ↓
[阶段 5] 提取章节、表格、图片 → 按类型分组
  ↓
[阶段 6] 分块（naive_merge）→ 生成合适大小的检索单元
  ↓
[阶段 7] Token 化和向量化 → 生成 embedding 并存储
  ↓
存储到 Elasticsearch/Infinity
```

## 核心特性

- **精确定位**：每个文本块都保留原始坐标信息（`x0`, `x1`, `top`, `bottom`），支持检索结果在原文档中高亮显示
- **结构识别**：自动识别标题、正文、表格、图片、页眉、页脚等不同布局类型
- **表格解析**：支持复杂表格结构识别（行列、单元格合并、表头识别），输出 HTML 格式
- **混合检索**：最终 Chunk 同时支持向量检索（768 维 embedding）和关键词检索（BM25）
- **多粒度分词**：生成 `content_ltks`（细粒度）、`content_sm_ltks`（粗粒度）、`important_kwd`（关键词）等多种检索字段

## 核心代码入口

- PDF 解析器：[deepdoc/parser/pdf_parser.py](file:///d:/work/rag/ragflow/deepdoc/parser/pdf_parser.py)
- 布局识别：[deepdoc/vision/layout_recognizer.py](file:///d:/work/rag/ragflow/deepdoc/vision/layout_recognizer.py)
- 表格识别：[deepdoc/vision/table_structure_recognizer.py](file:///d:/work/rag/ragflow/deepdoc/vision/table_structure_recognizer.py)
- 应用层编排：[rag/app/naive.py](file:///d:/work/rag/ragflow/rag/app/naive.py)
- 文本合并与分块：[rag/nlp/rag_tokenizer.py](file:///d:/work/rag/ragflow/rag/nlp/rag_tokenizer.py)
- 搜索与检索：[rag/nlp/search.py](file:///d:/work/rag/ragflow/rag/nlp/search.py)
- Chunk 数据结构：[api/apps/restful_apis/chunk_api.py](file:///d:/work/rag/ragflow/api/apps/restful_apis/chunk_api.py)
- Go 统一解析结果：[internal/parser/parser/parse_result.go](file:///d:/work/rag/ragflow/internal/parser/parser/parse_result.go)

## 与其他模块的关系

- **文档上传**：用户上传 PDF → 文件存储 → 解析任务入队
- **普通分块流程**：PDF 解析是分块流程的输入源，生成的 sections/tables 会被 chunk 规则进一步处理（参见 [ace/chunk-rules](file:///d:/work/rag/ragflow/ace/chunk-rules) 目录）
- **GraphRAG**：PDF 解析生成的 chunks 可作为 GraphRAG 的输入，用于实体关系抽取（参见 [ace/graphrag](file:///d:/work/rag/ragflow/ace/graphrag) 目录）
- **检索引擎**：最终 Chunks 存储在 Elasticsearch/Infinity，支持向量+关键词混合检索

## 数据流转示例

以一个 3 页产品手册为例：

1. **原始输入**：PDF 文件（3 页：封面+规格表+联系页）
2. **OCR 输出**：每页的文本框 + 坐标（如 `{"text": "智能手表产品手册", "x0": 120.5, ...}`）
3. **布局分析**：给每个框打上标签（`title`/`text`/`table`/`footer`）
4. **表格识别**：规格表被解析为行列结构 + HTML
5. **文本合并**：相邻段落合并为完整文本块
6. **分块**：控制每个 chunk 在 128-512 tokens 之间
7. **最终 Chunk**：包含 `content`、`q_768_vec`、`positions`、`table_html` 等字段，写入索引

## 相关文档

- 如需了解分块规则（naive/paper/book/qa/table/resume 等），请参阅 [ace/chunk-rules](file:///d:/work/rag/ragflow/ace/chunk-rules)
- 如需了解知识图谱增强检索，请参阅 [ace/graphrag](file:///d:/work/rag/ragflow/ace/graphrag)
