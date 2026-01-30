---
name: document-intelligence
description: |
  **[REQUIRED]** For **ALL** document/file extraction, parsing, or processing tasks with Snowflake Cortex AI.
  Use when: extracting data from PDFs, parsing documents/files, processing invoices/contracts/reports, 
  analyzing charts/blueprints, building document pipelines, reading files from stage, OCR tasks.
  Triggers: 
  - Documents: document extraction, parse document, documents in stage, docs, paperwork, papers, pages, forms, sheets
  - Files: file extraction, parse file, files in stage, process files, read PDF, attachments, uploads, assets, blobs, objects, artifacts, staged files, uploaded files
  - Formats: PDFs, Word docs, Excel files, spreadsheets, slides, presentations, images, scans, screenshots, scanned documents
  - Business docs: invoices, receipts, bills, statements, quotes, purchase orders, POs, contracts, agreements, policies, claims, reports, manuals, guides, specs, datasheets, applications, certificates, licenses, letters, memos, proposals, work orders, medical scans, lab results, prescriptions, medical records, patient records, healthcare documents, clinical notes, discharge summaries, insurance claims, EOBs, radiology reports, pathology reports
  - Actions: extract from file, extract info, get data from, pull info from, read text from, OCR, digitize, scrape, capture data, analyze these, parse these
  - Functions: AI_EXTRACT, AI_PARSE_DOCUMENT, AI_COMPLETE, document AI
---

# Document Intelligence

Entry point for all document processing tasks using Snowflake Cortex AI.

## Reference Files

This skill uses reference documentation for detailed function guidance:

| Reference | Location | Use For |
|-----------|----------|---------|
| AI_EXTRACT | `reference/ai-extract.md` | Function reference for structured extraction |
| AI_PARSE_DOCUMENT & AI_COMPLETE | `reference/ai-parse-doc-and-ai-complete.md` | Function reference for content parsing |
| **Extraction** | `reference/extraction.md` | Flow A: Structured field/table extraction workflow |
| **Parsing** | `reference/parsing.md` | Flow B: Full content parsing workflow |
| **Visual Analysis** | `reference/visual-analysis.md` | Flow C: Charts, blueprints, diagrams analysis |
| **Pipeline** | `reference/pipeline.md` | Post-processing: pipelines, storage, RAG integration |

**Load sub-skill based on user's extraction goal:**
- For structured fields/tables → Read `reference/extraction.md`
- For full text with layout or OCR → Read `reference/parsing.md`
- For charts, blueprints, diagrams → Read `reference/visual-analysis.md`
- After any flow completes → Read `reference/pipeline.md` for post-processing

## When to Use

- User wants to extract data from documents (PDFs, images, Office files)
- User mentions AI_EXTRACT or AI_PARSE_DOCUMENT
- User wants to process invoices, contracts, forms, reports
- User needs to analyze charts, blueprints, diagrams
- User wants to build document processing pipelines

## Workflow

```
Start
  ↓
Step 1: Gather Document Info
  ↓
  ├─→ Local file → PUT to Snowflake stage
  │
  ├─→ Other storage (GDrive, Dropbox, etc.) → Load openflow skill
  │                                              ↓
  │                                           Setup connector
  │                                              ↓
  │                                           Return here
  ↓
Step 2: Determine Extraction Goal
  ↓
  ├─→ Structured fields/tables → Load reference/extraction.md (Flow A)
  │
  ├─→ Full text with layout → Load reference/parsing.md (Flow B - LAYOUT)
  │
  ├─→ Text via OCR → Load reference/parsing.md (Flow B - OCR)
  │
  └─→ Images/charts/blueprints → Load reference/visual-analysis.md (Flow C)
  ↓
Step 3: Post-Processing Options
  ↓
  ├─→ Store results → Create table / Create pipeline
  │
  └─→ Build RAG/search integration
```

### Step 1: Gather Document Information

**Ask** user about their document using `ask_user_question`:

**Question 1 - File Format:**
```
What type of file do you want to process?
Options:
1. PDF
2. Image (PNG, JPEG, TIFF)
3. Office document (DOCX, PPTX)
4. HTML or TXT file
5. CSV file - AI_EXTRACT only
6. MD or EML file - AI_EXTRACT only
7. Excel file (XLSX, XLS) - not supported
8. Legacy Office (DOC, PPT) - needs conversion
```

**Supported Formats Summary:**
- **AI_EXTRACT supports:** PDF, images (PNG, JPEG, TIFF), DOCX, PPTX, HTML, TXT, CSV, MD, EML
- **AI_PARSE_DOCUMENT supports:** PDF, images (PNG, JPEG, TIFF), DOCX, PPTX, HTML, TXT

**If user selects "CSV file":** CSV files are supported by AI_EXTRACT only (not AI_PARSE_DOCUMENT). Continue to Step 2 to determine extraction goal.

**If user selects "MD or EML file":** MD and EML files are supported by AI_EXTRACT only (not AI_PARSE_DOCUMENT). Continue to Step 2 to determine extraction goal.

**If user selects "HTML or TXT file":** HTML and TXT files are supported by both AI_EXTRACT and AI_PARSE_DOCUMENT. Continue to Step 2 to determine extraction goal.

**If user selects "Legacy Office (DOC, PPT)":**
```
Legacy Office formats (DOC, PPT) are not directly supported.

Please convert to modern formats:
- DOC → DOCX (Save As in Word)
- PPT → PPTX (Save As in PowerPoint)
- Or export as PDF

Then upload the converted file and continue.
```

**If user selects "Excel file (XLSX, XLS)":**

Excel files are not directly supported by AI_EXTRACT or AI_PARSE_DOCUMENT. Inform the user of their options:

```
Excel files (.xlsx, .xls) are not directly supported by the document AI functions.

You have two options:

Option 1: Convert to PDF
- Export the Excel file as PDF from Excel/Google Sheets
- Then upload the PDF and use AI_EXTRACT or AI_PARSE_DOCUMENT

Option 2: Load directly into Snowflake (Recommended for tabular data)
- Snowflake has native Excel support for loading data into tables:

-- Infer schema from Excel file
SELECT * FROM TABLE(
  INFER_SCHEMA(
    LOCATION => '@my_stage/data.xlsx',
    FILE_FORMAT => 'TYPE=EXCEL'
  )
);

-- Query Excel file directly  
SELECT * FROM @my_stage/data.xlsx 
(FILE_FORMAT => 'TYPE=EXCEL', SHEET_NAME => 'Sheet1');

-- Load into table
COPY INTO my_table FROM @my_stage/data.xlsx
FILE_FORMAT = (TYPE = EXCEL)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

Which option would you like to proceed with?
```

If user chooses Option 1 (convert to PDF), guide them to upload the PDF and continue with the normal flow.

If user chooses Option 2 (load directly), help them with the Snowflake SQL commands above.

**Question 2 - File Location:**
```
Where is your file stored?
Options:
1. Internal Snowflake stage (e.g., @my_db.my_schema.my_stage/file.pdf)
2. External stage (S3, Azure Blob, GCS)
3. Local file on my computer
4. Other storage (Google Drive, Dropbox, SharePoint, OneDrive, Box)
5. Need help setting up a stage
```

**If user selects "Local file on my computer":**

Help user upload file to a Snowflake stage using PUT command.

1. **First, ensure a stage exists** (or create one):
```sql
-- Create a stage if needed (with server-side encryption)
CREATE STAGE IF NOT EXISTS my_db.my_schema.doc_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  COMMENT = 'Stage for document processing';
```

2. **Ask user for file path:**
```
What is the full path to your local file?
Example: /Users/username/Documents/invoice.pdf
         C:\Users\username\Documents\invoice.pdf
```

3. **Generate PUT command** (must be run via SnowSQL, Snowflake CLI, or Python connector):

```sql
-- Using SnowSQL or Snowflake CLI
PUT file:///path/to/your/file.pdf @my_db.my_schema.doc_stage AUTO_COMPRESS=FALSE;
```

**Provide instructions based on user's tool:**

**Option A - SnowSQL:**
```bash
# Install SnowSQL if not installed
# https://docs.snowflake.com/en/user-guide/snowsql-install-config

# Connect and upload
snowsql -a <account> -u <username>
PUT file:///path/to/file.pdf @my_db.my_schema.doc_stage AUTO_COMPRESS=FALSE;
```

**Option B - Snowflake CLI (snow):**
```bash
# Upload using Snowflake CLI
snow stage copy /path/to/file.pdf @my_db.my_schema.doc_stage --overwrite
```

**Option C - Python:**
```python
from snowflake.connector import connect

conn = connect(account='...', user='...', password='...')
cursor = conn.cursor()

# Upload file
cursor.execute("""
    PUT file:///path/to/file.pdf @my_db.my_schema.doc_stage 
    AUTO_COMPRESS=FALSE OVERWRITE=TRUE
""")

print("File uploaded successfully!")
conn.close()
```

**Option D - Multiple files:**
```sql
-- Upload all PDFs from a directory
PUT file:///path/to/folder/*.pdf @my_db.my_schema.doc_stage AUTO_COMPRESS=FALSE;
```

4. **Verify upload:**
```sql
-- List files in stage
LIST @my_db.my_schema.doc_stage;

-- Or using directory table
SELECT * FROM DIRECTORY(@my_db.my_schema.doc_stage);
```

5. **Refresh directory table** (required for DIRECTORY queries):
```sql
ALTER STAGE @my_db.my_schema.doc_stage REFRESH;
```

After file is uploaded, continue with extraction using the stage path.

**If user selects "Other storage":**
- Inform user that files need to be ingested into a Snowflake stage first
- **Load** the `openflow` skill to help set up a connector for their storage provider
- After openflow setup completes, return here to continue with extraction

**Question 3 - Content Type:**
```
What kind of content is in your document?
Options:
1. Structured forms (invoices, receipts, applications)
2. Reports/articles with text and tables
3. Charts, graphs, blueprints, diagrams
4. General text (manuals, policies, contracts)
```

### Step 2: Determine Extraction Goal

**Ask** user what they want to extract:

```
What do you want to extract?
Options:
1. Specific fields (names, dates, amounts, IDs)
2. Tables with defined columns
3. Full text with layout preservation (tables, headers, structure)
4. Text only via OCR (quick text extraction)
5. Images and visual content for analysis (charts, blueprints)
```

**Route based on response:**

| User Selection | Load Sub-Skill | Function |
|----------------|----------------|----------|
| Specific fields, Tables | `reference/extraction.md` | AI_EXTRACT |
| Full text with layout | `reference/parsing.md` | AI_PARSE_DOCUMENT (LAYOUT mode) |
| Text only via OCR | `reference/parsing.md` | AI_PARSE_DOCUMENT (OCR mode) |
| Images, Charts, Blueprints | `reference/visual-analysis.md` | AI_COMPLETE (vision) |

**After routing:** Load the appropriate sub-skill file and follow its workflow.

### Step 3: Post-Processing Options

After extraction/parsing/visual analysis is complete, **route to pipeline sub-skill**:

→ **Load `reference/pipeline.md`** to guide user through post-processing options:
- One-time extraction (done)
- Store results in Snowflake table
- Set up continuous processing pipeline
- Build RAG/search integration with vector embeddings

The pipeline sub-skill contains templates for all document processing patterns.

## Key Constraints

| Constraint | AI_EXTRACT | AI_PARSE_DOCUMENT |
|------------|------------|-------------------|
| Max file size | 100 MB | 50 MB |
| Max pages | 125 | 500 |
| Output format | JSON | Markdown |

## Stopping Points

- After Step 1 if file format is unsupported
- After Step 2 before routing to sub-skill
- After extraction completes for post-processing choice

## Output

Routes user to appropriate sub-skill with gathered context, then assists with post-processing.

---

## CRITICAL: Follow-Up Questions and New Extraction Requests

**ALWAYS re-evaluate the best flow when user asks a follow-up question or wants to extract something different.**

Do NOT assume the same flow applies. Each new request may require a different approach.

### When to Re-Evaluate

Re-evaluate the flow selection when user:
- Asks to extract different fields or data
- Wants to try a different approach
- Mentions a different document or file
- Asks follow-up questions about extraction
- Says "what about...", "can you also...", "now extract...", "try instead..."
- Expresses dissatisfaction with current results

### Re-Evaluation Process

**Ask** the user (or determine from context):

```
For this new request, let me determine the best approach.

What do you want to extract?
1. Specific fields or tables → Flow A (Extraction with AI_EXTRACT)
2. Full text with layout → Flow B (Parsing with AI_PARSE_DOCUMENT)
3. Visual content (charts, blueprints) → Flow C (Visual Analysis with AI_COMPLETE)
```

### Flow Selection Guide

| User Wants | Best Flow | Why |
|------------|-----------|-----|
| Named fields (invoice_number, date, vendor) | **Flow A: Extraction** | AI_EXTRACT returns structured JSON |
| Table data with known columns | **Flow A: Extraction** | AI_EXTRACT handles tables well |
| Full document text with structure | **Flow B: Parsing** | AI_PARSE_DOCUMENT preserves layout |
| OCR from scanned documents | **Flow B: Parsing** | AI_PARSE_DOCUMENT OCR mode |
| Chart/graph interpretation | **Flow C: Visual Analysis** | AI_COMPLETE vision understands images |
| Blueprint/diagram analysis | **Flow C: Visual Analysis** | AI_COMPLETE vision for complex visuals |
| Photo/screenshot analysis | **Flow C: Visual Analysis** | AI_COMPLETE vision for image content |

### Example Scenarios

**Scenario 1:** User extracted invoice fields (Flow A), now asks "can you also get the full text?"
→ Re-evaluate: Full text = **Flow B (Parsing)**, not Flow A

**Scenario 2:** User parsed document text (Flow B), now asks "extract just the table of line items"
→ Re-evaluate: Specific table = **Flow A (Extraction)**, not Flow B

**Scenario 3:** User extracted fields (Flow A), now asks "analyze the chart on page 3"
→ Re-evaluate: Chart analysis = **Flow C (Visual Analysis)**, not Flow A

**Scenario 4:** User analyzed a chart (Flow C), now asks "get all the data points as a table"
→ Stay on **Flow C** but use structured output prompt, OR switch to **Flow A** if chart is embedded in PDF

### Never Assume

❌ **Wrong:** "Since we used AI_EXTRACT before, let me continue with that..."
✅ **Right:** "For extracting the full text with layout, AI_PARSE_DOCUMENT (Flow B) is the better approach."

❌ **Wrong:** "I'll use the same approach for this new request..."
✅ **Right:** "Let me determine which flow is best for extracting chart data - that would be Visual Analysis (Flow C)."
