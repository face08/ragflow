# TXT 文本解析模块

## 模块概览

TXT 文本解析模块负责处理纯文本文档（.txt, .py, .js, .java, .c, .cpp, .h, .php, .go, .ts, .sh, .cs, .kt, .sql 等）的解析和分块。与 PDF 解析相比，TXT 解析流程更加简洁直接，专注于文本内容的编码检测、分隔符切分、段落合并和向量化。

**核心特点：**
- **无需 OCR**：文本已经是机器可读格式，跳过 OCR 阶段
- **无布局分析**：纯文本无需区分 title/text/table/figure 等区域类型
- **无表格识别**：不涉及行列结构和单元格合并
- **无位置坐标**：最终 Chunk 不包含 bbox 坐标和页码信息（无法支持原文高亮回溯）
- **编码自适应**：自动检测 UTF-8/GBK/GB18030 等编码格式
- **分隔符灵活**：支持自定义多字符分隔符（如 `\n!?;。；！？`）
- **段落合并智能**：基于 token 预算的贪婪或严格合并策略

**适用场景：**
- 代码文件（Python, JavaScript, Java, C/C++, Go, TypeScript 等）
- 纯文本配置文件（.txt, .conf, .ini）
- Shell 脚本（.sh, .bash）
- SQL 脚本
- 日志文件（需要自定义分隔符）
- 技术文档的纯文本版本

---

## 文档导航

### 解析流程详解
- **[01-完整解析流程.md](./01-完整解析流程.md)** - TXT 文本解析的完整 7 阶段流程，包含每个阶段的详细数据结构示例

---

## 核心特性

### 1. 编码检测与规范化
- **BOM 检测优先**：识别 UTF-8-BOM、UTF-16-LE/BE、UTF-32-LE/BE
- **UTF-8 优先策略**：先尝试 UTF-8 解码，失败后使用 chardet 自动检测
- **置信度阈值**：chardet 置信度 > 0.2 时采纳检测结果，否则回退到 "utf-8" 并忽略错误
- **GBK/GB2312 规范化**：统一转换为 GB18030（向下兼容 GBK 和 GB2312）
- **错误处理**：`errors='ignore'` 策略避免解码失败导致崩溃

### 2. 分隔符处理
- **换行符规范化**：所有 CRLF (`\r\n`) 统一转为 LF (`\n`)
- **多字符分隔符**：支持反引号包裹的多字符分隔符（如 <code>\`\\n\\n\`</code>）和裸字符序列（如 `\n!?;。；！？`）
- **正则编译**：分隔符字段解析为正则模式 `[<chars>]`，支持标点和换行符混合
- **空段落过滤**：`re.split()` 后自动过滤掉空字符串

### 3. 段落合并策略
- **MergeStrategy.OVER_CAP**（贪婪策略）：
  - 允许一次边界溢出（最后一个段落可以超出 token 预算）
  - 适用场景：需要保持段落完整性，避免过度切分
  - 示例：预算 128 tokens，最后一组可以达到 130 tokens

- **MergeStrategy.UNDER_CAP**（严格上限）：
  - 严格限制每组不超过 token 预算
  - 适用场景：需要精确控制 chunk 大小，避免向量化开销
  - 示例：预算 128 tokens，所有组都 ≤ 128 tokens

### 4. Sections 格式
- **统一格式**：`[[merged_text, ""]]`
  - 第一个元素：合并后的文本内容（多段落用换行符连接）
  - 第二个元素：空字符串（TXT 无需区分 text/table/image）
- **与 PDF 对比**：
  - PDF: `[[text_1, layout_type_1], [text_2, layout_type_2], ...]`
  - TXT: `[[text, ""]]`（简化版，无需布局类型）

### 5. 分词与向量化
- **content_ltks**：BM25 全文检索分词（中英文混合，停用词过滤）
- **content_sm_ltks**：短文本摘要分词（前 512 tokens）
- **q_768_vec**：768 维向量 embedding（用于向量检索）
- **混合检索**：向量检索 + BM25 全文检索，提升召回率

---

## 代码入口

### 主要文件路径
```
rag/app/naive.py (L1275-1280)       - TXT 解析入口，调用 TxtParser
deepdoc/parser/txt_parser.py        - TxtParser 类实现
rag/nlp/__init__.py                 - 核心解析函数库
  ├─ decode_text() (L200-228)       - 文本编码检测与解码
  ├─ find_codec() (L154-197)        - 编码格式猜测
  ├─ merge_paragraphs() (L1364-1393)- 段落合并主函数
  ├─ MergeStrategy (L1271-1283)     - 合并策略枚举
  └─ tokenize_chunks()              - 分词和向量化
rag/nlp/delim.py                    - 分隔符处理工具
  ├─ normalize_text_newlines()      - 换行符规范化
  ├─ parse_delimiter_field()        - 解析分隔符字段
  └─ compile_delimiter_pattern()    - 编译正则模式
api/apps/restful_apis/chunk_api.py  - 最终 Chunk 数据结构定义
```

### 调用链路
```
naive.py: chunk()
  └─> TxtParser().__call__(filename, binary, chunk_token_num, delimiter)
       ├─> decode_text(binary)  # 编码检测
       ├─> normalize_text_newlines(txt)  # 换行符规范化
       ├─> compile_delimiter_pattern(delimiter)  # 编译分隔符
       ├─> re.split(pattern, txt)  # 切分段落
       ├─> merge_paragraphs(paragraphs, chunk_token_num, strategy)  # 合并段落
       └─> [[merged_text, ""]]  # 生成 sections
  └─> _normalize_section_text_for_rtl_presentation_forms(sections)  # RTL 规范化
  └─> tokenize_chunks(Chunk)  # 分词向量化
```

---

## 与其他模块的关系

### 与 PDF 解析的对比

| 维度 | PDF 解析 | TXT 解析 |
|------|---------|---------|
| **复杂度** | 高（7 阶段：OCR + 布局 + 表格 + 合并 + 提取 + 分块 + 向量化） | 低（7 阶段：编码 + 规范化 + 切分 + 合并 + sections + RTL + 向量化） |
| **OCR 识别** | ✅ 必需（图片 PDF、扫描件） | ❌ 不需要（已是机器可读文本） |
| **布局分析** | ✅ 区分 title/text/table/figure | ❌ 无需布局类型 |
| **表格识别** | ✅ 行列结构、单元格合并、HTML 输出 | ❌ 无表格处理 |
| **位置坐标** | ✅ bbox (x0, x1, top, bottom) | ❌ 无坐标信息 |
| **页码信息** | ✅ page_num_int | ❌ 空数组 `[]` |
| **原文回溯** | ✅ 支持（positions 字段） | ❌ 不支持（positions 为空） |
| **编码处理** | ✅ OCR 输出已是 UTF-8 | ✅ 需要检测和转换（BOM、chardet） |
| **分隔符** | ❌ 由 OCR 和布局分析隐式确定 | ✅ 用户可自定义（`\n!?;。；！？`） |
| **段落合并** | ✅ XGBoost 模型预测 | ✅ 基于 token 预算的策略（OVER_CAP/UNDER_CAP） |
| **Sections 格式** | `[[text, layout_type], ...]` | `[[text, ""], ...]` |
| **最终 Chunk** | 包含坐标、页码、图片、表格 | 仅包含文本内容，无坐标和页码 |

### 与其他文档类型的关系
- **Markdown**：基于 TXT 解析，增加语法树解析（标题层级、代码块、列表、表格）
- **HTML**：基于 TXT 解析，增加 DOM 树解析和标签过滤
- **JSON**：基于 TXT 解析，增加 JSON 结构验证和扁平化
- **Excel/CSV**：独立解析路径，直接提取表格数据，不涉及 TXT 流程
- **DOCX**：混合路径，文本段落走类似 TXT 的流程，表格和图片独立提取

---

## 关键设计原则

1. **编码鲁棒性**：BOM 检测 + UTF-8 优先 + chardet 回退 + 忽略错误，保证各种编码文件都能正常解析
2. **分隔符灵活性**：支持多字符分隔符，适应代码、日志、配置文件等多样化场景
3. **段落完整性**：OVER_CAP 策略允许适度溢出，避免在段落中间硬切分
4. **统一接口**：最终输出 Chunk 格式与 PDF 一致，存储层无需区分文档类型
5. **性能优先**：跳过 OCR、布局、表格等重计算步骤，解析速度远快于 PDF

---

## 常见问题

### Q1: TXT 文档为什么没有原文高亮回溯功能？
**A**: TXT 解析后的 Chunk 不包含位置坐标（positions 字段为空），因为纯文本无需 OCR 和布局分析，缺少 bbox 坐标信息。如果需要高亮回溯，建议使用 Markdown 或 HTML 格式，保留结构化的锚点信息。

### Q2: 如何自定义 TXT 分隔符？
**A**: 在 `parser_config` 中设置 `delimiter` 参数：
```python
parser_config = {
    "chunk_token_num": 128,
    "delimiter": "\\n!?;。；！？"  # 换行符 + 句号 + 问号 + 感叹号（中英文）
}
```
支持两种语法：
- 裸字符序列：`\n!?;。；！？`（常用标点）
- 反引号包裹：<code>\`\\n\\n\`</code>（多字符分隔符，如双换行）

### Q3: OVER_CAP 和 UNDER_CAP 策略如何选择？
**A**: 
- **OVER_CAP（默认推荐）**：适用于大多数场景，允许最后一个段落适度溢出（例如预算 128 tokens，实际可能到 130 tokens），保持段落完整性，避免语义截断。
- **UNDER_CAP（严格上限）**：适用于向量化成本敏感的场景（如使用昂贵的 embedding 模型），严格限制每个 chunk 不超过 token 预算。

### Q4: TXT 解析支持哪些编码格式？
**A**: 
- **UTF-8**（优先）：包括 UTF-8-BOM
- **UTF-16/UTF-32**：LE/BE 字节序自动检测
- **GBK/GB2312**：自动规范化为 GB18030
- **其他编码**：通过 chardet 自动检测（置信度阈值 0.2）
- **未知编码**：回退到 UTF-8 + `errors='ignore'`，避免崩溃

### Q5: 代码文件（.py, .js, .java）解析效果如何？
**A**: 
- **优点**：保留完整的代码结构（函数、类、注释），支持多语言
- **缺点**：无法识别代码语法树（AST），仅按分隔符切分（建议使用 `\n\n` 双换行作为分隔符，按函数/类自然分块）
- **改进建议**：如果需要精确的函数级别检索，建议开发专门的代码解析器（基于 AST 的 tree-sitter 或 Language Server Protocol）

---

## 参考资料

- [RAGFlow 文档解析架构](../README.md)
- [PDF 解析流程](../pdf-parsing/01-完整解析流程.md)
- [Chunk 数据结构规则](../chunk-rules/chunk-schema.md)（如存在）
- [向量检索与 BM25](../chunk-rules/retrieval-strategies.md)（如存在）

---

**注意**：本文档基于 RAGFlow 当前代码库（2026-09-18）编写，后续版本可能会有 API 和实现细节的变化。
