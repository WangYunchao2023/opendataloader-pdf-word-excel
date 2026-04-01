---
name: opendataloader-pdf
version: 1.7.0
version_date: 2026-04-01
description: 文档解析工具（统一处理 PDF / Word / Excel），自动检测文档类型、智能选择模式（Fast/Hybrid）。Word 输入时采用「docx内容 + PDF位置」双来源合并；Excel 输入时采用「openpyxl结构化数据 + PDF页码」合并。统一输出 JSON（含完整数据、headers、data_rows 数组）和 Markdown，方便 AI 读取、分析、汇总与自动撰写。
---

# opendataloader-pdf SKILL

## 工具简介

基于 [OpenDataLoader PDF](https://github.com/opendataloader-project/opendataloader-pdf) 的文档解析工具，支持 **PDF / Word / Excel 统一处理**。**核心特性：自动检测文档类型并智能选择处理模式。**

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

---

## 🚀 快速开始（推荐）

使用 `opendataloader_auto.py` **自动检测 + 自动选择模式**，无需手动判断：

```bash
# 1. 自动检测并转换（智能选择 Fast / Hybrid 模式）
python3 opendataloader_auto.py document.pdf -o output/        # PDF
python3 opendataloader_auto.py document.docx -o output/      # Word（自动识别）

# 2. 仅检测类型，不转换
python3 opendataloader_auto.py document.pdf --detect-only   # PDF
python3 opendataloader_auto.py document.docx --detect-only   # Word

# 3. 强制使用某模式（仅 PDF）
python3 opendataloader_auto.py document.pdf -o output/ --force-mode hybrid

# 4. 停止 Hybrid server
python3 opendataloader_auto.py --stop-server
```

**自动检测流程（PDF）**：
1. 用 pypdf 读取前 5 页，统计有文字的页数
2. 有文字页 > 80% → **Fast（本地）模式**，无需启动 server
3. 有文字页 < 20% → **Hybrid + OCR 模式**，自动启动 server
4. 混合文档（20%~80%）→ **Hybrid + force-ocr + full 模式**，全部页面走 AI
5. 自动检测语言（中文/英文），用于 OCR 配置

**支持的文档格式**：

| 格式 | 处理方式 | 输出格式 | 位置追溯 |
|------|---------|---------|---------|
| PDF（数字/扫描/混合）| 自动路由（Fast 或 Hybrid）| Markdown + JSON | ✅ 完整（页码+bbox） |
| Word（.docx / .doc）| **自动转 PDF 再提取**（LibreOffice）| Markdown + JSON | ✅ 完整（页码+bbox） |
| Excel（.xlsx / .xls）| **openpyxl 直接提取 + Excel转PDF获取页码** | Markdown + JSON | ✅ 页码（sheet 级） |

---

## 前置要求

- **Java 11+**（必须，JAR 内置其中）
  - 路径：`/home/wangyc/opt/jre/amazon-corretto-11.0.30.7.1-linux-x64/bin/java`
  - 验证：`/home/wangyc/opt/jre/amazon-corretto-11.0.30.7.1-linux-x64/bin/java -version`
- Python 3.10+
- Python 包：`pypdf`（自动检测用）、`opendataloader-pdf`

```bash
pip install -U opendataloader-pdf "opendataloader-pdf[hybrid]" pypdf
```

---

## 模式说明

| 模式 | 适用场景 | Server | 速度 | 准确率 |
|------|---------|--------|------|--------|
| **Fast（本地）** | 标准数字 PDF（文字可复制）| ❌ 不需要 | 快（0.05s/页）| 中（0.72） |
| **Hybrid** | 扫描/混合/复杂文档 | ✅ 需要 | 中（0.43s/页）| 高（0.90） |

### 什么情况自动用 Hybrid？

自动路由脚本 `opendataloader_auto.py` 会根据以下规则自动选择：

- PDF 有 > 80% 页面含可提取文字 → **Fast 模式**（本地）
- PDF 有 < 20% 页面含文字（大部分扫描）→ **Hybrid + OCR**
- PDF 混合有文字页和扫描页 → **Hybrid + force-ocr + full 模式**
- 手动指定 `--force-mode hybrid` → **Hybrid 模式**

### 处理粒度

- **底层**：工具按页逐页处理（benchmark 0.43s/页）
- **用户感知**：一次性处理整个 PDF（或用 `--pages` 指定页码范围）
- **自动检测**：读取前 5 页判断类型，整文档应用同一模式

---

## 手动模式（可选，熟悉后可跳过）

如果想手动控制，按下表选择：

| 文档类型 | 模式 | Server 启动命令 | 调用命令 |
|----------|------|---------------|----------|
| **标准数字 PDF** | Fast（默认）| 无 | `opendataloader-pdf file.pdf -o out/` |
| **复杂/嵌套表格** | Hybrid | `opendataloader-pdf-hybrid --port 5002` | `opendataloader-pdf --hybrid docling-fast file.pdf -o out/` |
| **扫描版 PDF** | Hybrid + OCR | `opendataloader-pdf-hybrid --port 5002 --force-ocr` | `opendataloader-pdf --hybrid docling-fast file.pdf -o out/` |
| **非英文扫描 PDF** | Hybrid + OCR + 语言 | `opendataloader-pdf-hybrid --port 5002 --force-ocr --ocr-lang "zh,en"` | `opendataloader-pdf --hybrid docling-fast file.pdf -o out/` |
| **含数学公式** | Hybrid + Formula | `opendataloader-pdf-hybrid --port 5002 --enrich-formula` | `opendataloader-pdf --hybrid docling-fast --hybrid-mode full file.pdf -o out/` |
| **含图表需描述** | Hybrid + Picture | `opendataloader-pdf-hybrid --port 5002 --enrich-picture-description` | `opendataloader-pdf --hybrid docling-fast --hybrid-mode full file.pdf -o out/` |

### Hybrid Server 手动启动

```bash
export JAVA_HOME=/home/wangyc/opt/jre/amazon-corretto-11.0.30.7.1-linux-x64
opendataloader-pdf-hybrid --port 5002
```

**可选参数**：
- `--force-ocr` — 对所有页面强制 OCR（扫描 PDF）
- `--ocr-lang "zh,en"` — 指定 OCR 语言
- `--enrich-formula` — 提取数学公式（LaTeX）
- `--enrich-picture-description` — AI 生成图表描述
- `--hybrid-mode full` — 跳过智能分流，所有页面送 hybrid
- `--log-level debug` — 调试日志

---

## CLI 完整参数（opendataloader-pdf）

```bash
# 通用参数
opendataloader-pdf <file.pdf> -o <output_dir/>     # 指定输出目录
opendataloader-pdf <file.pdf> -f json,markdown      # 指定输出格式（逗号分隔）
opendataloader-pdf <file.pdf> --pages 1,3,5-7      # 指定页码
opendataloader-pdf <file.pdf> -q                    # 静默模式

# 本地模式（Fast，标准数字 PDF）
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

---

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
| **自动检测 + 自动路由（PDF）** | ✅ | Auto |
| **Word → PDF → 核心提取** | ✅ | 自动转换 |
| **内容指纹（章节路径/全局序号/内容预览）** | ✅ | JSON 含追溯字段 |
| **AI 精确回溯（原文档 + PDF页码 + bbox）** | ✅ | 完整链路 |
| **Excel 结构化提取（headers/data_rows/charts）** | ✅ | openpyxl |
| Auto-tagging → Tagged PDF | ⏳ Q2 2026 | — |

---

## 内容指纹与回溯（AI 友好）

### JSON 输出新增字段（PDF 和 Word 通用）

每个 `flat_elements` 中的元素都包含以下追溯字段：

| 字段 | 说明 | 示例 |
|------|------|------|
| `page number` | PDF 真实页码 | `3` |
| `bounding box` | 页面坐标 [left,bottom,right,top] | `[57.75, 669, 431, 713]` |
| `section_path` | 标题层级路径 | `统计分析报告 > 声明 > 签名` |
| `table_index` | 全文表格序号（仅表格）| `5` |
| `paragraph_index` | 全文段落序号（仅段落）| `12` |
| `content_preview` | 内容前100字（用于快速定位）| `临床研究负责单位：...` |
| `original_word` | 原始 Word 文件路径（Word→PDF 时）| `/path/to/report.docx` |
| `converted_pdf` | 转换后的 PDF 路径（Word→PDF 时）| `/tmp/report.pdf` |

### 顶层元数据

```json
{
  "doc_type": "word-to-pdf",
  "original_word": "/path/to/report.docx",
  "converted_pdf": "/tmp/report.pdf",
  "traceability": {
    "total_elements": 1263,
    "total_paragraphs": 871,
    "total_tables": 70,
    "total_images": 3
  }
}
```

### Excel JSON 输出结构

```json
{
  "doc_type": "excel",
  "source": "openpyxl",
  "sheets": ["Sheet1", "明细", "汇总"],
  "total_sheets": 3,
  "total_tables": 5,
  "total_charts": 2,
  "elements": [
    {
      "type": "table",
      "sheet": "明细",
      "table_index": 1,
      "headers": ["部门", "岗位", "人数", "AI应用比例"],
      "data_rows": [
        ["研发部", "临床监察", 5, 0.65],
        ["医学部", "医学写作", 3, 0.80]
      ],
      "row_count": 8,
      "col_count": 4,
      "section_path": "Excel > 文件名 > 明细"
    },
    {
      "type": "chart",
      "sheet": "汇总",
      "chart_type": "BarChart",
      "chart_title": "各部门AI应用比例",
      "content": "[BarChart] 各部门AI应用比例 | 数据范围: Sheet1!B2:D10"
    }
  ]
}
```

> **Excel 的 `data_rows` 是结构化二维数组**，AI 可以直接做数据分析、图表重建、汇总计算，无需二次解析。

### AI 回溯示例

当 AI 处理这些数据后，可以精确回答：

> "该不良事件发生率数据位于 **原始报告第 4 页**，
> 坐标 bbox=[50,600,560,720]，
> 属于「统计分析报告 > 所有不良事件 > 原始结果」章节，
> 为全文第 12 张表格（table_index=12）。"



| 问题 | 解决方案 |
|------|----------|
| "java not found" | 确认 JAVA_HOME 设置正确，或使用 `opendataloader_auto.py` 自动处理 |
| 扫描 PDF 未提取 | 用 `opendataloader_auto.py` 自动检测，会自动启用 OCR |
| 混合 PDF（扫描+文字）| `opendataloader_auto.py` 会自动用 Hybrid + force-ocr + full |
| OCR 中文不工作 | auto 模式会自动检测中文；手动模式 server 加 `--ocr-lang "zh,en"` |
| 表格提取不准 | 用 `--table-method cluster`，或 Hybrid 模式 |
| 公式未提取 | 手动 Hybrid 模式 server 加 `--enrich-formula` |
| 图表没有描述 | 手动 Hybrid 模式 server 加 `--enrich-picture-description` |
| 重复调用慢 | 批量传入多个文件（每次调用会启动 JVM），或保持 Hybrid server 运行 |
| 混合 PDF 文字页没被识别 | 用 `--force-mode hybrid` 强制 Hybrid，或 `--force-mode fast` 强制 Fast |

---

## LangChain 集成

```bash
pip install langchain-opendataloader-pdf
```

```python
from langchain_opendataloader_pdf import OpenDataLoaderPDF

loader = OpenDataLoaderPDF(file_path="document.pdf")
docs = loader.load()
```

---

## 资源链接

- [GitHub](https://github.com/opendataloader-project/opendataloader-pdf)
- [官方文档](https://opendataloader.org/docs/quick-start-python)
- [Hybrid 模式指南](https://opendataloader.org/docs/hybrid-mode)
- [JSON Schema](https://opendataloader.org/docs/json-schema)
- [LangChain 集成](https://docs.langchain.com/oss/python/integrations/document_loaders/opendataloader_pdf)
