---
title: Agent Skills 最佳實踐
date: 2026-04-22 22:21:00
categories: AI
tags:
  - Claude
  - Agent Skills
  - Best Practices
---
本文檔總結了 Claude 官方發布的 Agent Skills 編寫最佳實踐，幫助你建立高效、可維護的技能。
<!-- more-->

## 核心原則

### 簡潔至上

上下文視窗是公共資源。你的技能與 Claude 需要知道的所有內容共享上下文視窗，包括：
- 系統提示
- 對話歷史
- 其他技能的元數據
- 你的實際請求

**預設假設：** Claude 已經非常聰明

只需要添加 Claude 不知道的上下文。質疑每一條資訊：
- "Claude 真的需要這個解釋嗎？"
- "我可以假設 Claude 知道這個嗎？"
- "這個段落值得消耗的 token 嗎？"

**好的示例 - 簡潔**（約 50 tokens）：
```markdown
## Extract PDF text

Use pdfplumber for text extraction:

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
```

**差的示例 - 過於冗長**（約 150 tokens）：
```markdown
## Extract PDF text

PDF (Portable Document Format) files are a common file format that contains
text, images, and other content. To extract text from a PDF, you'll need to
use a library. There are many libraries available for PDF processing, but
pdfplumber is recommended because it's easy to use and handles most cases well.
First, you'll need to install it using pip. Then you can use the code below...
```

簡潔版本假設 Claude 知道什麼是 PDF 以及庫的工作原理。

### 設定適當的自由度

根據任務的脆弱性和可變性匹配具體程度：

**高自由度**（基於文字的指令）：

適用於：
- 多種方法都有效
- 決策取決於上下文
- 啟發式方法指導方法

**中自由度**（帶參數的偽代碼或腳本）：

適用於：
- 存在首選模式
- 可以接受一些變化
- 配置影響行為

**低自由度**（特定腳本，幾乎無參數）：

適用於：
- 操作脆弱且容易出錯
- 一致性很關鍵
- 必須遵循特定順序

### 使用所有計劃使用的模型進行測試

技能作為模型的附加功能，因此效果取決於底層模型。在所有計劃使用的模型上測試你的技能。

**按模型的測試考慮：**
- **Claude Haiku**（快速、經濟）：技能是否提供足夠的指導？
- **Claude Sonnet**（平衡）：技能是否清晰高效？
- **Claude Opus**（強大推理）：技能是否避免過度解釋？

對 Opus 完美的方案可能需要更多細節才能適用於 Haiku。如果計劃跨多個模型使用技能，請在所有模型上都能良好運行的說明。

## 技能結構

### 命名約定

使用一致的命名模式使技能更容易引用和討論。考慮使用 **動名詞形式**（動詞 + -ing）作為技能名稱。

技能名稱欄位要求：
- 最多 64 個字元
- 只能使用小寫字母、數字和連字符
- 不能包含 XML 標籤
- 不能包含保留詞："anthropic"、"claude"

**好的命名示例（動名詞形式）：**
- `processing-pdfs`
- `analyzing-spreadsheets`
- `managing-databases`
- `testing-code`
- `writing-documentation`

### 編寫有效的描述

`description` 欄位支援技能發現，應包括技能做什麼以及何時使用它。

**始終使用第三人稱**。描述被注入系統提示，不一致的人稱可能導致發現問題。

- **好的：** "Processes Excel files and generates reports"
- **避免：** "I can help you process Excel files"
- **避免：** "You can use this to process Excel files"

**具體並包含關鍵詞**。包括技能做什麼以及具體的觸發條件/上下文。

**有效的示例：**

```yaml
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

### 漸進式披露模式

SKILL.md 作為概述，將 Claude 指向所需的詳細材料，就像入職指南中的目錄。

**實用指導：**
- 保持 SKILL.md 正文在 500 行以下以獲得最佳效能
- 接近此限制時，將內容拆分到單獨的文件中

#### 模式 1：帶引用的高級指南

```markdown
---
name: pdf-processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF Processing

## Quick start

Extract text with pdfplumber:
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## Advanced features

**Form filling**: See [FORMS.md](FORMS.md) for complete guide
**API reference**: See [REFERENCE.md](REFERENCE.md) for all methods
**Examples**: See [EXAMPLES.md](EXAMPLES.md) for common patterns

#### 模式 2：按領域組織

對於具有多個領域的技能，按領域組織內容以避免載入無關上下文。

```text
bigquery-skill/
├── SKILL.md (overview and navigation)
└── reference/
    ├── finance.md (revenue, billing metrics)
    ├── sales.md (opportunities, pipeline)
    ├── product.md (API usage, features)
    └── marketing.md (campaigns, attribution)
```

#### 模式 3：條件細節

顯示基本內容，連結到高級內容：

```markdown
# DOCX Processing

## Creating documents

Use docx-js for new documents. See [DOCX-JS.md](DOCX-JS.md).

## Editing documents

For simple edits, modify the XML directly.

**For tracked changes**: See [REDLINING.md](REDLINING.md)
**For OOXML details**: See [OOXML.md](OOXML.md)
```

### 避免深度巢狀引用

Claude 可能在從其他引用檔案引用時部分讀取檔案。保持引用從 SKILL.md 深度為一层。

### 為較長的引用檔案添加目錄

對於超過 100 行的引用檔案，在頂部包含目錄。

## 工作流和回饋循環

### 為複雜任務使用工作流

將複雜操作分解為清晰的順序步驟。對於特別複雜的工作流，提供一個清單供 Claude 複製到回應中並在進展時勾選。

**示例：研究綜合工作流**

```markdown
## Research synthesis workflow

Copy this checklist and track your progress:

```
Research Progress:
- [ ] Step 1: Read all source documents
- [ ] Step 2: Identify key themes
- [ ] Step 3: Cross-reference claims
- [ ] Step 4: Create structured summary
- [ ] Step 5: Verify citations
```
```

### 實現回饋循環

**常見模式：** 運行驗證器 → 修復錯誤 → 重複

此模式大大提高輸出品質。

```markdown
## Content review process

1. Draft your content following the guidelines in STYLE_GUIDE.md
2. Review against the checklist:
   - Check terminology consistency
   - Verify examples follow the standard format
   - Confirm all required sections are present
3. If issues found:
   - Note each issue with specific section reference
   - Revise the content
   - Review the checklist again
4. Only proceed when all requirements are met
5. Finalize and save the document
```

## 內容指南

### 避免時間敏感資訊

不要包含會過時的資訊：

**好的示例**（使用 "old patterns" 部分）：
```markdown
## Current method

Use the v2 API endpoint: `api.example.com/v2/messages`

## Old patterns

<details>
<summary>Legacy v1 API (deprecated 2025-08)</summary>

The v1 API used: `api.example.com/v1/messages`

This endpoint is no longer supported.
</details>
```

### 使用一致的術語

選擇一個術語並在整個技能中使用它：

**好 - 一致：**
- 始終使用 "API endpoint"
- 始終使用 "field"
- 始終使用 "extract"

**差 - 不一致：**
- 混合使用 "API endpoint"、"URL"、"API route"、"path"
- 混合使用 "field"、"box"、"element"、"control"
- 混合使用 "extract"、"pull"、"get"、"retrieve"

## 常見模式

### 模板模式

為輸出格式提供模板。根據需要匹配嚴格程度。

### 示例模式

對於輸出品質取決於看到示例的技能，提供輸入/輸出對：

```markdown
## Commit message format

Generate commit messages following these examples:

**Example 1:**
Input: Added user authentication with JWT tokens
Output:
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```
```

### 條件工作流模式

指導 Claude 穿過決策點：

```markdown
## Document modification workflow

1. Determine the modification type:

   **Creating new content?** → Follow "Creation workflow" below
   **Editing existing content?** → Follow "Editing workflow" below

2. Creation workflow:
   - Use docx-js library
   - Build document from scratch
   - Export to .docx format

3. Editing workflow:
   - Unpack existing document
   - Modify XML directly
   - Validate after each change
   - Repack when complete
```

## 評估和迭代替換

### 先建立評估

**在編寫大量文件之前建立評估。** 這確保你的技能解決實際問題而不是記錄想像的問題。

**評估驅動的開發：**
1. **識別差距：** 在沒有技能的情況下運行 Claude 代表性任務
2. **建立評估：** 構建三個測試這些差距的場景
3. **建立基線：** 測量沒有技能的 Claude 效能
4. **編寫最小指令：** 建立足夠的內容來解決差距並通過評估
5. **迭代：** 執行評估，與基線比較，然後改進

### 使用 Claude 迭代開發技能

最有效的技能開發過程涉及 Claude 本身。與一個 Claude 實例（"Claude A"）合作，建立被其他實例（"Claude B"）使用的技能。

**迭代現有技能：**
1. **在真實工作流中使用技能：** 給 Claude B（載入了技能）實際任務
2. **觀察 Claude B 的行為：** 注意它在哪裡掙扎、成功或做出意外選擇
3. **回到 Claude A 進行改進：** 分享當前的 SKILL.md 並描述你觀察到的內容

## 反模式要避免

### 避免 Windows 風格路徑

始終在檔案路徑中使用正斜杠，即使在 Windows 上：

- ✓ **好：** `scripts/helper.py`、`reference/guide.md`
- ✗ **避免：** `scripts\helper.py`、`reference\guide.md`

### 避免提供太多選項

不要呈現多種方法，除非必要：

```
**差示例：太多選擇**（令人困惑）：
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**好示例：提供預設**（帶出路）：
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
```

## 高級：帶可執行代碼的技能

### 解決，不要放棄

為技能編寫腳本時，處理錯誤條件而不是放棄給 Claude。

**好的示例：明確處理錯誤：**

```python
def process_file(path):
    """Process a file, creating it if it doesn't exist."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # Create file with default content instead of failing
        print(f"File {path} not found, creating default")
        with open(path, "w") as f:
            f.write("")
        return ""
    except PermissionError:
        # Provide alternative instead of failing
        print(f"Cannot access {path}, using default")
        return ""
```

**差的示例：放棄給 Claude：**

```python
def process_file(path):
    # Just fail and let Claude figure it out
    return open(path).read()
```

### 提供實用腳本

即使 Claude 可以編寫腳本，預製腳本也有優勢：

- 比生成的代碼更可靠
- 節省 token（無需在上下文中包含代碼）
- 節省時間（無需生成代碼）
- 確保使用一致性

### 使用視覺分析

當輸入可以渲染為圖像時，讓 Claude 分析它們：

```markdown
## Form layout analysis

1. Convert PDF to images:
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. Analyze each page image to identify form fields
3. Claude can see field locations and types visually
```

### 建立可驗證的中間輸出

當 Claude 執行複雜、開放性任務時，它可能會犯錯。"計畫-驗證-執行"模式通過讓 Claude 首先以結構化格式建立計畫，然後在執行之前用腳本驗證該計畫來及早發現錯誤。

### 執行環境

技能在具有檔案系統存取、bash 命令和代碼執行功能的代碼執行環境中運行。

**Claude 如何存取技能：**

1. **元數據預載入：** 在啟動時，所有技能的 YAML frontmatter 中的名稱和描述被載入到系統提示中
2. **檔案按需讀取：** Claude 使用 bash Read 工具在需要時從檔案系統存取 SKILL.md 和其他檔案
3. **腳本高效執行：** 實用腳本可以通過 bash 執行，而無需將其完整內容載入到上下文中。只有腳本的輸出消耗 token
4. **大檔案無上下文懲罰：** 引用檔案、資料或文件在實際讀取之前不消耗上下文 token

### MCP 工具引用

如果你的技能使用 MCP（模型上下文協定）工具，始終使用完全限定的工具名稱以避免"工具未找到"錯誤。

**格式：** `ServerName:tool_name`

**示例：**
```markdown
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

### 避免假設工具已安裝

不要假設套件可用：

```
**差的示例：假設安裝**：
"Use the pdf library to process the file."

**好的示例：明確依賴**：
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```
```

## 有效技能的檢查清單

### 核心品質
- [ ] 描述具體且包含關鍵詞
- [ ] 描述包括技能做什麼以及何時使用它
- [ ] SKILL.md 正文在 500 行以下
- [ ] 詳細資訊在單獨的檔案中（如需要）
- [ ] 無時間敏感資訊（或在 "old patterns" 部分）
- [ ] 整個過程一致的術語
- [ ] 示例具體，而非抽象
- [ ] 檔案引用深度為一层
- [ ] 適當使用漸進式披露
- [ ] 工作流有清晰的步驟

### 代碼和腳本
- [ ] 腳本解決問題而不是放棄給 Claude
- [ ] 錯誤處理明確且有幫助
- [ ] 無"巫毒常數"（所有值都有理由）
- [ ] 所需套件在說明中列出並驗證可用
- [ ] 腳本有清晰的文檔
- [ ] 無 Windows 風格路徑（全部使用正斜杠）
- [ ] 關鍵操作有驗證/驗證步驟
- [ ] 品質關鍵任務包含回饋循環

### 測試
- [ ] 至少建立三個評估
- [ ] 使用 Haiku、Sonnet 和 Opus 測試
- [ ] 使用真實使用場景測試
- [ ] 如適用，納入團隊回饋