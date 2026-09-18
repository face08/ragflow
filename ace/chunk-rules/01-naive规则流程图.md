# Naive通用切片规则流程图

## 规则概述

**Naive规则**是RAGFlow中最核心的通用切片模板,支持多种文档格式(PDF、DOCX、Excel、Markdown、HTML、TXT、EPUB、JSON等),是处理绝大多数文档类型的默认选择。

## 核心特性

- **多格式支持**: 支持10+种文档格式
- **多解析器后端**: 支持10种PDF解析器(DeepDOC、MinerU、MonkeyOCRv2、Docling、OpenDataLoader、TCADP、PaddleOCR、Somark、Mistral OCR、PlainText)
- **灵活的切片策略**: 基于token数量的朴素合并(naive_merge)
- **重叠支持**: 支持chunk间的重叠以保持上下文连贯性
- **表格和图片上下文注入**: 可配置表格和图片的上下文大小

## 主流程图

```mermaid
flowchart TD
    Start([开始: chunk函数调用]) --> CheckParams[检查参数<br/>filename, binary, from_page, to_page, lang, kwargs]
    CheckParams --> InitConfig[初始化parser_config<br/>- chunk_token_num: 512<br/>- delimiter: DEFAULT_DELIMITER<br/>- layout_recognize: DeepDOC<br/>- overlapped_percent: 0.2]
    
    InitConfig --> DetectFormat{检测文件格式}
    
    DetectFormat -->|PDF| PDFPath[PDF处理路径]
    DetectFormat -->|DOCX| DOCXPath[DOCX处理路径]
    DetectFormat -->|Excel| ExcelPath[Excel处理路径]
    DetectFormat -->|Markdown| MarkdownPath[Markdown处理路径]
    DetectFormat -->|HTML| HTMLPath[HTML处理路径]
    DetectFormat -->|TXT| TXTPath[TXT处理路径]
    DetectFormat -->|EPUB| EPUBPath[EPUB处理路径]
    DetectFormat -->|JSON| JSONPath[JSON处理路径]
    
    %% PDF处理路径
    PDFPath --> DispatchPDFParser[调度PDF解析器<br/>根据layout_recognize选择]
    DispatchPDFParser --> PDFParserChoice{选择解析器}
    
    PDFParserChoice -->|DeepDOC| DeepDOCParser[DeepDOC解析器<br/>深度学习布局识别]
    PDFParserChoice -->|MinerU| MinerUParser[MinerU解析器<br/>挖掘式解析]
    PDFParserChoice -->|MonkeyOCRv2| MonkeyParser[MonkeyOCRv2解析器<br/>OCR识别]
    PDFParserChoice -->|Docling| DoclingParser[Docling解析器<br/>文档理解]
    PDFParserChoice -->|OpenDataLoader| ODLParser[OpenDataLoader解析器<br/>开放数据加载]
    PDFParserChoice -->|TCADP| TCADPParser[TCADP解析器<br/>腾讯云文档解析]
    PDFParserChoice -->|PaddleOCR| PaddleParser[PaddleOCR解析器<br/>百度OCR]
    PDFParserChoice -->|Somark| SomarkParser[Somark解析器<br/>标记解析]
    PDFParserChoice -->|Mistral OCR| MistralParser[Mistral OCR解析器<br/>Mistral视觉模型]
    PDFParserChoice -->|PlainText| PlainTextParser[PlainText解析器<br/>纯文本提取]
    
    DeepDOCParser --> PDFExtractSections[提取PDF章节<br/>带布局信息]
    MinerUParser --> PDFExtractSections
    MonkeyParser --> PDFExtractSections
    DoclingParser --> PDFExtractSections
    ODLParser --> PDFExtractSections
    TCADPParser --> PDFExtractSections
    PaddleParser --> PDFExtractSections
    SomarkParser --> PDFExtractSections
    MistralParser --> PDFExtractSections
    PlainTextParser --> PDFExtractSections
    
    PDFExtractSections --> AddTableContext{是否注入表格上下文?<br/>table_context_size > 0}
    AddTableContext -->|是| InjectTableCtx[注入表格上下文<br/>前后N个chunk]
    AddTableContext -->|否| AddImageContext
    InjectTableCtx --> AddImageContext{是否注入图片上下文?<br/>image_context_size > 0}
    AddImageContext -->|是| InjectImageCtx[注入图片上下文<br/>前后N个chunk]
    AddImageContext -->|否| MergeSections
    InjectImageCtx --> MergeSections
    
    %% DOCX处理路径
    DOCXPath --> ExtractDocxParas[提取DOCX段落<br/>识别标题层级]
    ExtractDocxParas --> DetectDocxTitles[检测标题<br/>基于样式和格式]
    DetectDocxTitles --> BuildDocxHierarchy[构建层级结构<br/>text + title]
    BuildDocxHierarchy --> MergeSections
    
    %% Excel处理路径
    ExcelPath --> ParseExcelSheets[解析Excel工作表<br/>逐sheet处理]
    ParseExcelSheets --> ExtractTables[提取表格数据<br/>DataFrame格式]
    ExtractTables --> ProcessTableImages[处理表格中的图片<br/>Vision模型描述]
    ProcessTableImages --> MergeSections
    
    %% Markdown处理路径
    MarkdownPath --> ParseMarkdown[解析Markdown<br/>识别标题层级]
    ParseMarkdown --> ExtractMDStructure[提取结构<br/># ## ### 等]
    ExtractMDStructure --> MergeSections
    
    %% HTML处理路径
    HTMLPath --> ParseHTML[解析HTML<br/>Beautiful Soup]
    ParseHTML --> ExtractHTMLText[提取文本<br/>保留结构]
    ExtractHTMLText --> CleanHTML[清理HTML标签]
    CleanHTML --> MergeSections
    
    %% TXT处理路径
    TXTPath --> ReadText[读取纯文本]
    ReadText --> SplitByDelimiter[按分隔符分割<br/>DEFAULT_DELIMITER]
    SplitByDelimiter --> MergeSections
    
    %% EPUB处理路径
    EPUBPath --> ParseEPUB[解析EPUB<br/>提取章节]
    ParseEPUB --> ExtractEPUBChapters[提取章节内容<br/>保留结构]
    ExtractEPUBChapters --> MergeSections
    
    %% JSON处理路径
    JSONPath --> ParseJSON[解析JSON]
    ParseJSON --> ExtractJSONContent[提取内容字段]
    ExtractJSONContent --> MergeSections
    
    %% 合并章节
    MergeSections[调用naive_merge合并章节<br/>参数: sections, chunk_token_num, delimiter, overlapped_percent]
    
    MergeSections --> NaiveMergeDetail[naive_merge详细流程<br/>见核心函数流程图]
    
    NaiveMergeDetail --> TokenizeChunks[调用tokenize_chunks<br/>转换为最终文档格式]
    
    TokenizeChunks --> AddMetadata[添加元数据<br/>- docnm_kwd: 文件名<br/>- title_tks: 标题tokens<br/>- content_with_weight: 内容<br/>- content_ltks: 内容tokens<br/>- important_kwd: 重要关键词<br/>- positions: 位置信息]
    
    AddMetadata --> ReturnChunks[返回chunks列表]
    ReturnChunks --> End([结束])
    
    style Start fill:#e1f5e1
    style End fill:#ffe1e1
    style PDFPath fill:#e3f2fd
    style DOCXPath fill:#f3e5f5
    style ExcelPath fill:#fff3e0
    style MarkdownPath fill:#e8f5e9
    style HTMLPath fill:#fce4ec
    style TXTPath fill:#f1f8e9
    style EPUBPath fill:#e0f2f1
    style JSONPath fill:#fff9c4
```

## 配置参数说明

### parser_config参数

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| chunk_token_num | int | 512 | 每个chunk的目标token数量 |
| delimiter | str | "\n!?;。;!?~" | 文本分割分隔符 |
| layout_recognize | str | "DeepDOC" | PDF布局识别后端 |
| overlapped_percent | float | 0.2 | chunk重叠百分比(0-1) |
| table_context_size | int | 0 | 表格上下文大小(前后chunk数) |
| image_context_size | int | 0 | 图片上下文大小(前后chunk数) |
| analyze_hyperlink | bool | True | 是否分析超链接 |

### 支持的PDF解析器

1. **DeepDOC** (默认): 深度学习布局识别,准确度高
2. **MinerU**: 挖掘式解析,适合复杂布局
3. **MonkeyOCRv2**: OCR识别,适合扫描文档
4. **Docling**: 文档理解,适合学术论文
5. **OpenDataLoader**: 开放数据加载器
6. **TCADP**: 腾讯云文档解析,云端处理
7. **PaddleOCR**: 百度OCR,中文优化
8. **Somark**: 标记解析器
9. **Mistral OCR**: Mistral视觉模型
10. **PlainText**: 纯文本提取,最快但精度低

## 关键代码片段

### 主入口函数

```python
def chunk(filename, binary=None, from_page=0, to_page=MAXIMUM_PAGE_NUMBER, 
          lang="Chinese", callback=None, **kwargs):
    """
    主切片函数,根据文件格式调度到不同的解析器
    
    参数:
        filename: 文件名
        binary: 二进制内容(可选)
        from_page: 起始页码
        to_page: 结束页码
        lang: 语言("Chinese"或"English")
        callback: 进度回调函数
        **kwargs: 包含parser_config的额外参数
    
    返回:
        List[Dict]: chunk列表,每个chunk是一个字典
    """
    parser_config = kwargs.get("parser_config", {
        "chunk_token_num": 512,
        "delimiter": DEFAULT_DELIMITER,
        "layout_recognize": "DeepDOC"
    })
    
    # 格式检测和解析器选择
    eng = lang.lower() == "english"
    doc = {
        "docnm_kwd": filename,
        "title_tks": rag_tokenizer.tokenize(re.sub(r"\.[a-zA-Z]+$", "", filename))
    }
    
    # ... 根据文件扩展名调度到相应解析器 ...
    
    # 使用naive_merge合并章节
    chunks = naive_merge(sections, 
                        int(parser_config.get("chunk_token_num", 128)),
                        parser_config.get("delimiter", DEFAULT_DELIMITER),
                        overlapped_percent)
    
    # 转换为最终文档格式
    res.extend(tokenize_chunks(chunks, doc, eng, pdf_parser, language=lang))
    return res
```

### PDF解析器调度

```python
PARSERS = {
    "deepdoc": "by_deepdoc",
    "mineru": "by_mineru",
    "monkeyocrv2": "by_monkeyocrv2",
    "docling": "by_docling",
    "opendataloader": "by_opendataloader",
    "tcadp": "by_tcadp",
    "paddleocr": "by_paddleocr",
    "somark": "by_somark",
    "mistral ocr": "by_mistral",
    "plaintext": "by_plaintext"
}

def _dispatch_pdf_parser(parser_config):
    """根据配置调度到相应的PDF解析器"""
    layout_recognize = parser_config.get("layout_recognize", "DeepDOC").lower()
    parser_name = PARSERS.get(layout_recognize, "by_deepdoc")
    return getattr(PdfParser, parser_name)
```

## 使用场景

### 适用场景

1. **通用文档处理**: 当不确定文档类型时的默认选择
2. **混合格式文档**: 包含文本、表格、图片的复杂文档
3. **多语言文档**: 支持中英文及其他语言
4. **长文档**: 需要按token数量精确控制chunk大小
5. **需要上下文重叠**: 通过overlapped_percent保持语义连贯

### 不适用场景

1. **学术论文**: 使用paper规则更好,能提取标题/作者/摘要
2. **结构化Q&A**: 使用qa规则,专门提取问答对
3. **纯表格数据**: 使用table规则,每行作为一个chunk
4. **简历文档**: 使用resume规则,结构化提取简历信息

## 性能优化建议

1. **选择合适的解析器**: 
   - 质量优先: DeepDOC、Docling
   - 速度优先: PlainText
   - 扫描文档: PaddleOCR、MonkeyOCRv2

2. **调整chunk_token_num**:
   - 较小(128-256): 精确检索,但可能切断语义
   - 中等(512): 平衡的默认值
   - 较大(1024+): 保持完整上下文,但检索粒度粗

3. **设置合理的重叠**:
   - 0: 无重叠,速度最快
   - 0.2: 推荐值,适度重叠
   - 0.5+: 大量重叠,增加存储但提高召回

4. **上下文注入**: 仅在表格/图片对理解至关重要时启用

## 相关函数

- [naive_merge](./核心函数流程图.md#naive_merge): 朴素合并策略
- [tokenize_chunks](./核心函数流程图.md#tokenize_chunks): 文档格式化
- [add_positions](./核心函数流程图.md#add_positions): 位置元数据添加
