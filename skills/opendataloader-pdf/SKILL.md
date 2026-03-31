---
name: opendataloader-pdf
version: 1.1.0
version_date: 2026-03-31
description: PDF 解析工具，支持 OCR、表格提取、公式提取、图表分析，输出 Markdown/JSON/HTML 格式。用于 RAG 管道、PDF 内容提取、文档结构化。
---

# opendataloader-pdf SKILL

## 工具简介

基于 [OpenDataLoader PDF](https://github.com/opendataloader-project/opendataloader-pdf) 的 PDF 解析工具，支持 AI-ready 数据提取。**本地模式即装即用，Hybrid 模式需先启动服务器。**

**核心能力**：
- ✅ 文本提取（正确阅读顺序）
- ✅ 边界框（bbox）定位每个元素
- ✅ 表格提取（简单/复杂无边框）
- ✅ OCR（80+语言，扫描 PDF）
- ✅ 公式提取（LaTeX）
- ✅ 图表 AI 描述
- ✅ 输出：Markdown / JSON（带边界框）/ HTML

**准确率 Benchmark**：
| 引擎 | 总体准确率 | 表格 | 速度 |
|------|-----------|------|------|
| opendataloader [hybrid] | 0.90 | 0.93 | 0.43s/页 |
| opendataloader (local) | 0.72 | 0.49 | 0.05s/页 |
| docling | 0.86 | 0.89 | 0.73s/页 |
| marker | 0.83 | 0.81 | 53.93s/页 |

## 前置要求

- **Java 11+**（必须，JAR 内置其中）
  - 路径：`~/opt/jre/amazon-corretto-11.0.30.7.1-linux-x64/bin/java`
  - 环境变量：`export JAVA_HOME=~/opt/jre/amazon-corretto-11.0.30.7.1-linux-x64`
- Python 3.10+

## ⚠️ 重要：模式选择规则

**工具不会自动判断、自动切换模式。** 使用前需根据文档类型选择对应模式：

| 文档类型 | 模式 | Server 是否需要 | 启动命令 | 调用命令 |
|----------|------|----------------|----------|----------|
| **标准数字 PDF**（文字可复制） | **Fast（默认）** | ❌ 不需要 | 无 | `opendataloader-pdf file.pdf -o out/` |
| **复杂/嵌套表格** | **Hybrid** | ✅ 需要 | `opendataloader-pdf-hybrid --port 5002` | `opendataloader-pdf --hybrid docling-fast file.pdf -o out/` |
| **扫描版 PDF** | **Hybrid + OCR** | ✅ 需要 | `opendataloader-pdf-hybrid --port 5002 --force-ocr` | `opendataloader-pdf --hybrid docling-fast file.pdf -o out/` |
| **非英文扫描 PDF**（如中文） | **Hybrid + OCR + 语言** | ✅ 需要 | `opendataloader-pdf-hybrid --port 5002 --force-ocr --ocr-lang "zh,en"` | `opendataloader-pdf --hybrid docling-fast file.pdf -o out/` |
| **含数学公式的 PDF** | **Hybrid + Formula** | ✅ 需要 | `opendataloader-pdf-hybrid --port 5002 --enrich-formula` | `opendataloader-pdf --hybrid docling-fast --hybrid-mode full file.pdf -o out/` |
| **含图表需描述的 PDF** | **Hybrid + Picture** | ✅ 需要 | `opendataloader-pdf-hybrid --port 5002 --enrich-picture-description` | `opendataloader-pdf --hybrid docling-fast --hybrid-mode full file.pdf -o out/` |
| 无标签 PDF 生成 Tagged PDF | **Auto-tagging** | — | ⏳ Q2 2026 | — |

---

### Hybrid Server 启动（Hybrid 模式必须）

```bash
# 在终端 A 启动（需要一直运行）
export JAVA_HOME=~/opt/jre/amazon-corretto-11.0.30.7.1-linux-x64
opendataloader-pdf-hybrid --port 5002
```

**可选参数**：
- `--force-ocr` — 对所有页面强制 OCR（扫描 PDF）
- `--ocr-lang "zh,en"` — 指定 OCR 语言（默认自动检测）
- `--enrich-formula` — 提取数学公式（LaTeX）
- `--enrich-picture-description` — AI 生成图表描述
- `--hybrid-mode full` — 跳过智能分流，所有页面送 hybrid
- `--log-level debug` — 调试日志

## 安装命令

```bash
# 基础版（本地模式）
pip install -U opendataloader-pdf

# Hybrid AI 模式（支持 OCR/复杂表格/公式/图表描述）
pip install -U "opendataloader-pdf[hybrid]"
```

## 使用方式

### Python

```python
import opendataloader_pdf

opendataloader_pdf.convert(
    input_path=["file1.pdf", "file2.pdf", "folder/"],
    output_dir="output/",
    format="markdown,json"  # 支持: json, text, html, pdf, markdown, markdown-with-html, markdown-with-images
)
```

### CLI 完整参数

```bash
# 通用参数
opendataloader-pdf <file.pdf> -o <output_dir/>     # 指定输出目录
opendataloader-pdf <file.pdf> -f json,markdown      # 指定输出格式（逗号分隔）
opendataloader-pdf <file.pdf> --pages 1,3,5-7      # 指定页码
opendataloader-pdf <file.pdf> -q                    # 静默模式

# 本地模式参数（Fast，标准数字 PDF）
opendataloader-pdf <file.pdf> -o out/              # 即装即用，无需 server

# Hybrid 模式参数（复杂文档）
opendataloader-pdf --hybrid docling-fast <file.pdf> -o out/
opendataloader-pdf --hybrid docling-fast --hybrid-mode full <file.pdf>  # 全部页面走 hybrid

# 表格提取
opendataloader-pdf <file.pdf> --table-method cluster  # 复杂无边框表格

# 输出格式
opendataloader-pdf <file.pdf> --to-stdout           # 输出到终端（单格式）
```

### 输出格式说明

| 格式 | 说明 |
|------|------|
| `json` | 结构化 JSON，每个元素含 bbox（边界框）、type、page |
| `markdown` | 纯文本，带标题层级 |
| `markdown-with-html` | Markdown 内嵌 HTML（保留格式） |
| `markdown-with-images` | Markdown 内嵌图片引用 |
| `html` | 富文本 HTML |
| `text` | 纯文本（无结构） |
| `pdf` | 带标注的 PDF（显示边界框） |

### JSON 输出结构

```json
{
  "type": "paragraph",       // paragraph, heading, table, image, formula 等
  "content": "文本内容",
  "bbox": [left, bottom, right, top],  // PDF 坐标点（顺时针）
  "page": 1,
  "heading_level": 1        // 标题级别（仅 heading 类型有）
}
```

## 能力矩阵

| 功能 | 支持 | 模式 |
|------|------|------|
| 文本提取 + 正确阅读顺序 | ✅ | 本地 |
| 每个元素带边界框（bbox）| ✅ | 本地 |
| 简单表格提取（边框表格）| ✅ | 本地 |
| 复杂/无边框表格 | ✅ | Hybrid |
| 标题层级检测 | ✅ | 本地 |
| 列表检测（有序/无序/嵌套）| ✅ | 本地 |
| 图片提取 + 坐标 | ✅ | 本地 |
| AI 图表描述 | ✅ | Hybrid |
| 扫描 PDF OCR（80+ 语言）| ✅ | Hybrid |
| 公式提取（LaTeX）| ✅ | Hybrid |
| Tagged PDF 结构提取 | ✅ | 本地 |
| AI 安全过滤（防注入）| ✅ | 本地 |
| 页眉/页脚/水印过滤 | ✅ | 本地 |
| Auto-tagging → Tagged PDF | ⏳ Q2 2026 | — |

## 常见问题

| 问题 | 解决方案 |
|------|----------|
| "java not found" | 设置 `JAVA_HOME`，见上方环境变量 |
| 扫描 PDF 未提取 | 用 Hybrid 模式，启动 server 时加 `--force-ocr` |
| OCR 中文不工作 | server 启动加 `--ocr-lang "zh,en"` |
| 表格提取不准 | 用 `--table-method cluster`，或 Hybrid 模式 |
| 公式未提取 | server 加 `--enrich-formula`，调用加 `--hybrid-mode full` |
| 图表没有描述 | server 加 `--enrich-picture-description`，调用加 `--hybrid-mode full` |
| 重复调用慢 | 批量传入多个文件（每次调用会启动 JVM）|

## LangChain 集成

```bash
pip install langchain-opendataloader-pdf
```

```python
from langchain_opendataloader_pdf import OpenDataLoaderPDF

loader = OpenDataLoaderPDF(file_path="document.pdf")
docs = loader.load()
```

## 资源链接

- [GitHub](https://github.com/opendataloader-project/opendataloader-pdf)
- [官方文档](https://opendataloader.org/docs/quick-start-python)
- [Hybrid 模式指南](https://opendataloader.org/docs/hybrid-mode)
- [JSON Schema](https://opendataloader.org/docs/json-schema)
- [LangChain 集成](https://docs.langchain.com/oss/python/integrations/document_loaders/opendataloader_pdf)
