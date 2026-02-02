---
name: document-intelligence-coco-desktop
description: |
  **[REQUIRED]** For **ALL** document/file extraction, parsing, or processing tasks with Snowflake Cortex AI.
  Use when: extracting data from PDFs, parsing documents/files, processing invoices/contracts/reports, 
  analyzing charts/blueprints, building document pipelines, reading files from stage, OCR tasks.
  Triggers: 
  - Documents: document extraction, parse document, documents in stage, docs, paperwork, papers, pages, forms, sheets
  - Files: file extraction, parse file, files in stage, process files, read PDF, attachments, uploads, assets, blobs, objects, artifacts, staged files, uploaded files, file in stage, files on stage
  - Formats: PDFs, Word docs, Excel files, spreadsheets, slides, presentations, images, scans, screenshots, scanned documents
  - Business docs: invoices, receipts, bills, statements, quotes, purchase orders, POs, contracts, agreements, policies, claims, reports, manuals, guides, specs, datasheets, applications, certificates, licenses, letters, memo, proposals, work orders, medical scans, lab results, prescriptions, medical records, patient records, healthcare documents, clinical notes, discharge summaries, insurance claims, EOBs, radiology reports, pathology reports
  - Visual content: charts, graphs, diagrams, blueprints, flowcharts, floor plans, schematics, drawings, plots, bar chart, line chart, pie chart, infographic, visual analysis, analyze chart, analyze graph, analyze diagram
  - Actions: extract from file, extract info, get data from, pull info from, read text from, OCR, digitize, scrape, capture data, analyze these, parse these, summarize, summary of, analyze file, analyze document, analyze image, what is in this file, read this file
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
| **Pipeline** | `reference/pipeline.md` | Post-processing: pipelines, storage |
| **Google Drive Download** | `reference/google-drive-download.md` | Automated bulk download from Google Drive folders |

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
Step 1: Determine Extraction Goal (FIRST)
  ↓
  Analyze user request → May trigger multiple flows
  ↓
  ├─→ Structured fields/tables → Flow A (AI_EXTRACT)
  ├─→ Full text with layout/OCR → Flow B (AI_PARSE_DOCUMENT)
  ├─→ Charts/diagrams/blueprints → Flow C (AI_COMPLETE)
  └─→ Multiple needs → Multiple flows in sequence
  ↓
Step 2: Get File Location
  ↓
  ├─→ Snowflake stage → Proceed to extraction
  ├─→ Local file → Help upload via PUT command
  └─→ Other storage → Load openflow skill for connector
  ↓
Step 3: Infer File Type & Validate
  ↓
  Extract extension from file path → Check compatibility
  ↓
Step 4: Execute Flow(s)
  ↓
Step 5: Post-Processing Options
  ↓
  └─→ Load reference/pipeline.md
```

### Step 1: Determine Extraction Goal (ASK FIRST)

**Ask** user what they want to extract. Provide both options AND allow free-form response:

```
What would you like to extract or analyze from your document?

Options:
1. Structured extraction - Extract specific fields, tables, or data points (e.g., invoice number, line items)
2. Full text parsing - Get complete document text, with or without layout preserved (OCR for scans)
3. Visual analysis - Analyze charts, graphs, diagrams, blueprints, or other visual content
4. Multiple types - Combine any of the above (e.g., extract fields AND analyze a chart)

Or describe what you need in your own words:
- "Extract invoice number, vendor name, and total amount"
- "Get all the text content with formatting preserved"
- "Analyze the charts and extract data points"
- "Pull out the table of line items"
- "Get a summary of the document"
- "Extract specific fields AND get the full text"

What do you need?
```

### Analyzing the Request

**Carefully analyze** the user's response to determine which flow(s) to trigger:

| User Mentions | Flow to Trigger | Function |
|---------------|-----------------|----------|
| Specific fields (names, dates, amounts, IDs) | **Flow A** | AI_EXTRACT |
| Tables with columns | **Flow A** | AI_EXTRACT |
| Full text, all content, complete document | **Flow B** | AI_PARSE_DOCUMENT |
| Layout, formatting, structure preserved | **Flow B** | AI_PARSE_DOCUMENT |
| OCR, scanned document, text recognition | **Flow B** | AI_PARSE_DOCUMENT |
| Charts, graphs, plots, trends | **Flow C** | AI_COMPLETE |
| Diagrams, blueprints, schematics | **Flow C** | AI_COMPLETE |
| Visual analysis, what's in the image | **Flow C** | AI_COMPLETE |

### Multiple Flows Detection

**Check if multiple flows are needed:**

| User Request Pattern | Flows Needed |
|---------------------|--------------|
| "Extract fields AND get full text" | Flow A + Flow B |
| "Get invoice data AND analyze the chart" | Flow A + Flow C |
| "Parse the document AND extract the diagram" | Flow B + Flow C |
| "Extract everything - fields, text, and charts" | Flow A + Flow B + Flow C |

**If multiple flows detected:**
```
I notice you need multiple types of extraction:
- [List detected needs]

I'll process these in sequence:
1. [First flow]
2. [Second flow]
...

Let's start with [first flow]. After that completes, I'll proceed to the next.
```

### Step 2: Get File Location

#### Detect CoCo Desktop Environment First

**Check if running in CoCo Desktop with browser capability:**

Use browser_tabs() tool to detect browser availability.

**If CoCo Desktop detected:**
- Check for open browser tabs with Google Drive (drive.google.com)
- If Google Drive folder is open → Offer automated bulk download (see below)

**Ask** user where their file is stored:

```
Where is your file located?
Options:
1. Snowflake stage (e.g., @my_db.my_schema.my_stage/file.pdf)
2. Local file on my computer
3. Google Drive folder (bulk download) - [CoCo Desktop only]
4. External stage (S3, Azure Blob, GCS)
5. Cloud storage connector (Google Drive, Dropbox, SharePoint, OneDrive, Box)
```

**If Snowflake stage:** Get the full stage path and proceed.

**If local file:** Help upload using PUT command:

```sql
-- Create a stage if needed
CREATE STAGE IF NOT EXISTS my_db.my_schema.doc_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');

-- Upload file (run via SnowSQL, Snowflake CLI, or Python)
PUT file:///path/to/file.pdf @my_db.my_schema.doc_stage AUTO_COMPRESS=FALSE;

-- Verify upload
LIST @my_db.my_schema.doc_stage;
```

**If Google Drive folder (bulk download):** Load `reference/google-drive-download.md` for automated workflow:
- Detects if browser already has Google Drive folder open
- If not, asks for Google Drive folder URL
- Automates: Select all (Cmd+A) → Right-click Download → Upload to Snowflake stage
- Returns list of staged file paths for processing

**If cloud storage connector:** Load the `openflow` skill to set up a connector, then return here.

### Step 3: Infer File Type from Extension

**Do NOT ask user about file type.** Infer from the file path extension:

```
File path: @stage/invoices/invoice_001.pdf
Extension: .pdf → PDF file (supported by all functions)
```

**Extension to Format Mapping:**

| Extension | Format | AI_EXTRACT | AI_PARSE_DOCUMENT | AI_COMPLETE |
|-----------|--------|------------|-------------------|-------------|
| .pdf | PDF | ✅ | ✅ | ✅ (convert to image) |
| .png, .jpg, .jpeg, .tiff, .bmp, .gif, .webp | Image | ✅ | ✅ | ✅ |
| .docx | Word | ✅ | ✅ | ❌ |
| .pptx | PowerPoint | ✅ | ✅ | ❌ |
| .html, .htm | HTML | ✅ | ✅ | ❌ |
| .txt | Text | ✅ | ✅ | ❌ |
| .csv | CSV | ✅ | ❌ | ❌ |
| .md | Markdown | ✅ | ❌ | ❌ |
| .eml | Email | ✅ | ❌ | ❌ |
| .xlsx, .xls | Excel | ❌ | ❌ | ❌ |
| .doc, .ppt | Legacy Office | ❌ | ❌ | ❌ |

**If unsupported format detected:**

For Excel (.xlsx, .xls):
```
Excel files are not supported by document AI functions.

Options:
1. Export as PDF and re-upload
2. Load directly into Snowflake (recommended for tabular data):

SELECT * FROM @stage/file.xlsx (FILE_FORMAT => 'TYPE=EXCEL');
```

For Legacy Office (.doc, .ppt):
```
Legacy Office formats require conversion.
Please save as .docx/.pptx or export as PDF.
```

### Step 4: Execute Flow(s)

Based on extraction goal from Step 1, load the appropriate sub-skill(s):

| Flow | Sub-Skill | When |
|------|-----------|------|
| Flow A | `reference/extraction.md` | Structured fields, tables |
| Flow B | `reference/parsing.md` | Full text, layout, OCR |
| Flow C | `reference/visual-analysis.md` | Charts, diagrams, blueprints |

**For multiple flows:** Execute sequentially, completing each before starting the next.

### Step 5: Post-Processing Options

After extraction/parsing/visual analysis is complete, **route to pipeline sub-skill**:

→ **Load `reference/pipeline.md`** to guide user through post-processing options:
- One-time extraction (done)
- Store results in Snowflake table
- Set up continuous processing pipeline

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

Options:
1. Structured extraction - Specific fields or tables (AI_EXTRACT)
2. Full text parsing - Complete text, with or without layout (AI_PARSE_DOCUMENT)
3. Visual analysis - Charts, blueprints, diagrams (AI_COMPLETE)
4. Multiple types - Combine approaches

Or describe what you need in your own words.
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
