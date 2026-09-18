# RAGFlow Chunk 规则文档索引

本目录包含 RAGFlow 所有 chunk 切片规则的详细技术文档，包括流程图、源码分析和数据结构说明。

---

## 📚 文档导航

### 核心文档

- **[Chunk 输出数据结构说明](./chunk输出数据结构说明.md)** ⭐
  - 统一的 chunk 数据格式定义
  - 14 个规则的字段差异对比
  - 完整数据示例和使用建议
  - 核心函数实现和 Schema 映射

---

## 🔧 14 个 Chunk 规则详解

### 通用文本规则

| 规则 | 适用场景 | 核心特性 | 文档链接 |
|------|---------|---------|---------|
| **01. Naive** | 通用文档 | 基础递归切片 + 智能合并 | [流程图](./01-naive规则流程图.md) |
| **02. Paper** | 学术论文 | 章节树构建 + 作者信息 | [流程图](./02-paper规则流程图.md) |
| **03. Book** | 图书/长文档 | 深层目录树 + 层级合并 | [流程图](./03-book规则流程图.md) |
| **04. QA** | 问答对 | 固定分隔符切分 | [流程图](./04-qa规则流程图.md) |
| **06. Resume** | 简历 | 章节识别 + 层级切片 | [流程图](./06-resume规则流程图.md) |
| **07. Manual** | 技术手册 | 三级目录树 + 图片提取 | [流程图](./07-manual规则流程图.md) |
| **08. Laws** | 法律文档 | 条款树构建 + 法条解析 | [流程图](./08-laws规则流程图.md) |
| **09. Presentation** | 演示文稿 | 逐页切片 + 图片关联 | [流程图](./09-presentation规则流程图.md) |
| **13. Email** | 邮件 | 发件人/主题提取 | [流程图](./13-email规则流程图.md) |

### 结构化数据规则

| 规则 | 适用场景 | 核心特性 | 文档链接 |
|------|---------|---------|---------|
| **05. Table** | Excel/CSV | 逐行解析 + 表格渲染图 | [流程图](./05-table规则流程图.md) |
| **15. Tag** | 标签知识库 | 问题-标签映射 | [流程图](./15-tag规则流程图.md) |

### 多媒体规则

| 规则 | 适用场景 | 核心特性 | 文档链接 |
|------|---------|---------|---------|
| **10. One** | 单页 PDF | 整页切片 + 完整截图 | [流程图](./11-one规则流程图.md) |
| **11. Picture** | 图片文件 | OCR 识别 + 图片嵌入 | [流程图](./12-picture规则流程图.md) |
| **12. Audio** | 音频/视频 | 语音转录（无位置信息）| [流程图](./14-audio规则流程图.md) |

---

## 📊 规则特性对比

### 1. 切片策略

| 切片方式 | 使用规则 | 说明 |
|---------|---------|------|
| **递归分隔符** | Naive, Resume, Manual, Laws, Presentation, Email | 使用 `\n!?;。;！?\n` 等多级分隔符递归切分 |
| **章节树构建** | Paper, Book | 基于章节标题构建层级树，然后合并 |
| **固定分隔符** | QA | 使用 `\n` 切分问答对 |
| **逐行解析** | Table, Tag | 按 Excel 行或标签条目逐个处理 |
| **逐页切片** | One, Presentation | 每页作为独立 chunk |
| **OCR 识别** | Picture | 对图片进行文字识别 |
| **语音转录** | Audio | ASR 转录后切片 |

### 2. 合并策略

| 合并方式 | 使用规则 | 函数 |
|---------|---------|------|
| **Naive Merge** | Naive, QA, Resume, Email | `naive_merge()` - 简单按 token 数合并 |
| **Hierarchical Merge** | Paper, Book, Manual, Laws | `hierarchical_merge()` - 层级聚合 + RAPTOR 支持 |
| **Tree Merge** | Presentation | `tree_merge()` - 章节树合并 |
| **无合并** | Table, Tag, One, Picture, Audio | 直接输出原始 chunks |

### 3. 特殊字段支持

| 字段 | 使用规则 | 说明 |
|------|---------|------|
| `toc_kwd` | Paper, Book, Manual, Laws | 目录层级路径 |
| `authors_tks` | Paper | 论文作者分词 |
| `tag_kwd` | Tag | 标签关键词列表 |
| `image` | Table, One, Picture | 图片数据 |
| `doc_type_kwd` | Table, Picture | "table" 或 "image" |
| `position_int` | 除 Audio/Tag 外所有规则 | 位置坐标信息 |
| `raptor_kwd` | 支持 `raptor=True` 的规则 | RAPTOR 聚类标识 |

---

## 🔑 核心技术概念

### 1. 分词层级

所有规则都通过 `tokenize()` 函数生成三级分词：

```python
{
    "content_with_weight": "原始文本",      # 原始内容
    "content_ltks": "粗 粒度 分词",        # 用于 rag-coarse analyzer
    "content_sm_ltks": "细 粒 度 分 词"    # 用于 rag-fine analyzer
}
```

### 2. 位置追踪

PDF/DOCX 文档通过 `pdf_parser.crop()` 提取位置信息：

```python
{
    "position_int": [[page, left, right, top, bottom], ...],
    "page_num_int": [page1, page2, ...],
    "top_int": [top1, top2, ...]
}
```

### 3. RAPTOR 聚类

启用 `raptor=True` 后，`hierarchical_merge()` 会递归聚类生成多层级 chunk：

```python
{
    "raptor_kwd": "doc123_cluster5",
    "raptor_layer_int": 1  # 0=叶子节点, 1/2/...=聚类层
}
```

### 4. 层级切片

使用 `child_delimiters_pattern` 时，子 chunk 会保留父内容：

```python
{
    "content_with_weight": "1.1 小节内容...",
    "mom_with_weight": "第一章 引言\n..."  # 父 chunk 内容
}
```

---

## 🗂️ 源码文件映射

| 源码文件 | 包含规则 | 说明 |
|---------|---------|------|
| `rag/app/naive.py` | Naive | 基础切片规则 |
| `rag/app/paper.py` | Paper | 学术论文规则 |
| `rag/app/book.py` | Book | 图书规则 |
| `rag/app/qa.py` | QA | 问答对规则 |
| `rag/app/table.py` | Table | 表格规则 |
| `rag/app/resume.py` | Resume | 简历规则 |
| `rag/app/manual.py` | Manual | 技术手册规则 |
| `rag/app/laws.py` | Laws | 法律文档规则 |
| `rag/app/presentation.py` | Presentation | 演示文稿规则 |
| `rag/app/one.py` | One | 单页 PDF 规则 |
| `rag/app/picture.py` | Picture | 图片规则 |
| `rag/app/audio.py` | Audio | 音频规则 |
| `rag/app/email.py` | Email | 邮件规则 |
| `rag/app/tag.py` | Tag | 标签规则 |
| `rag/nlp/__init__.py` | - | 核心分词和合并函数 |

---

## 📖 使用指南

### 如何选择合适的规则？

1. **学术论文** → Paper（章节树 + 作者信息）
2. **技术书籍/长文档** → Book（深层目录树）
3. **技术手册/API 文档** → Manual（三级目录 + 图片）
4. **法律文档** → Laws（条款树解析）
5. **演示文稿/PPT** → Presentation（逐页切片）
6. **Excel/CSV 表格** → Table（逐行解析 + 表格图）
7. **FAQ/问答库** → QA（固定分隔符）
8. **简历** → Resume（章节识别）
9. **邮件** → Email（发件人/主题提取）
10. **扫描图片/截图** → Picture（OCR 识别）
11. **单页 PDF** → One（整页截图）
12. **音频/视频** → Audio（语音转录）
13. **标签知识库（Excel）** → Tag（问题-标签映射）
14. **通用文档** → Naive（智能递归切片）

### 关键配置参数

```python
parser_config = {
    "chunk_token_count": 128,           # chunk 目标 token 数
    "layout_recognize": True,           # 启用版面分析
    "raptor": False,                    # 启用 RAPTOR 聚类
    "html4excel": False,                # Excel 表格转 HTML
    "delimiter": "\n!?;。;！?\n",       # 递归分隔符
    "task_page_size": 12                # 任务页大小
}
```

---

## 🔗 相关资源

- **Elasticsearch Schema**: `conf/infinity_mapping.json`
- **分词器配置**: `rag/nlp/rag_tokenizer.py`
- **PDF 解析器**: `deepdoc/parser/`
- **OCR 引擎**: `deepdoc/vision/`

---

**文档维护**: RAGFlow 开发团队  
**最后更新**: 2026-09-18  
**版本**: v1.0
