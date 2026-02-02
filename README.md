# Document Intelligence Skills for Snowflake Cortex AI

Extract and parse information from documents and files using Snowflake's Cortex AI functions.

## Overview

A single comprehensive skill for all document/file processing needs:

| Skill | Purpose |
|-------|---------|
| `document-intelligence` | Complete document/file workflow - extraction, parsing, visual analysis, pipelines, evaluation |

## Trigger Words

The skill is triggered by mentions of:

| Category | Trigger Words |
|----------|---------------|
| **Documents** | document, docs, paperwork, papers, pages, forms, sheets |
| **Files** | file, attachments, uploads, assets, blobs, objects, artifacts, staged files, file in stage, files on stage |
| **Formats** | PDF, Word doc, Excel, spreadsheet, slides, presentation, image, scan, screenshot |
| **Business Docs** | invoice, receipt, bill, statement, quote, PO, contract, agreement, policy, claim, report, manual, guide, spec, datasheet, application, certificate, license, letter, memo, proposal, work order, medical scan, lab result, prescription, medical record, patient record, healthcare document, clinical note, discharge summary, insurance claim, EOB, radiology report, pathology report |
| **Visual Content** | charts, graphs, diagrams, blueprints, flowcharts, floor plans, schematics, drawings, plots, bar chart, line chart, pie chart, infographic, visual analysis |
| **Actions** | extract, parse, OCR, digitize, scrape, capture data, read text from, get data from, pull info from, summarize, summary of, analyze file, analyze chart, what is in this file |
| **Functions** | AI_EXTRACT, AI_PARSE_DOCUMENT, AI_COMPLETE |

## Directory Structure

```
document-intelligence-coco-desktop/
├── SKILL.md                                    # Main skill (entry point)
├── README.md                                   # This file
└── reference/
    ├── ai-extract.md                           # AI_EXTRACT function reference
    ├── ai-parse-doc-and-ai-complete.md         # AI_PARSE_DOCUMENT & AI_COMPLETE reference
    ├── extraction.md                           # Flow A: Structured field/table extraction
    ├── parsing.md                              # Flow B: Full content parsing
    ├── visual-analysis.md                      # Flow C: Charts, blueprints, diagrams
    ├── pipeline.md                             # Post-processing: pipelines, storage
    └── google-drive-download.md                # Google Drive bulk download automation
```

## Sub-Skills

| Sub-Skill | File | Purpose |
|-----------|------|---------|
| **Extraction** | `reference/extraction.md` | Flow A: AI_EXTRACT for structured fields/tables |
| **Parsing** | `reference/parsing.md` | Flow B: AI_PARSE_DOCUMENT for full text with layout |
| **Visual Analysis** | `reference/visual-analysis.md` | Flow C: AI_COMPLETE for charts, blueprints, diagrams |
| **Pipeline** | `reference/pipeline.md` | Post-processing: storage, pipelines |

### Flow Selection

The skill automatically re-evaluates which flow is best for each request:

| User Wants | Best Flow | Function |
|------------|-----------|----------|
| Named fields (invoice_number, date, vendor) | **Flow A: Extraction** | AI_EXTRACT |
| Table data with known columns | **Flow A: Extraction** | AI_EXTRACT |
| Full document text with structure | **Flow B: Parsing** | AI_PARSE_DOCUMENT |
| OCR from scanned documents | **Flow B: Parsing** | AI_PARSE_DOCUMENT |
| Chart/graph interpretation | **Flow C: Visual Analysis** | AI_COMPLETE |
| Blueprint/diagram analysis | **Flow C: Visual Analysis** | AI_COMPLETE |

**Important:** The skill re-evaluates the best flow for every follow-up question or new extraction request. It does not assume the same flow applies.

## Cortex AI Functions

| Function | Purpose | Max Size | Max Pages | Output |
|----------|---------|----------|-----------|--------|
| `AI_EXTRACT` | Structured field/table extraction | 100 MB | 125 | JSON |
| `AI_PARSE_DOCUMENT` | Full content parsing | 50 MB | 500 | Markdown |
| `AI_COMPLETE` | Visual analysis (charts, diagrams) | - | - | Text/JSON |

### Supported File Formats

| Format | AI_EXTRACT | AI_PARSE_DOCUMENT | Notes |
|--------|------------|-------------------|-------|
| PDF | ✅ | ✅ | Full support |
| PNG | ✅ | ✅ | Full support |
| JPEG/JPG | ✅ | ✅ | Full support |
| TIFF/TIF | ✅ | ✅ | Full support |
| Word (DOCX) | ✅ | ✅ | Full support |
| PowerPoint (PPTX) | ✅ | ✅ | Full support |
| HTML | ✅ | ✅ | Full support |
| TXT | ✅ | ✅ | Full support |
| CSV | ✅ | ❌ | AI_EXTRACT only |
| MD, EML | ✅ | ❌ | AI_EXTRACT only |
| Excel (XLSX/XLS) | ❌ | ❌ | Convert to PDF or load directly into Snowflake |
| DOC, PPT (legacy) | ❌ | ❌ | Convert to DOCX/PPTX or PDF |

See [official documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/parse-document#input-requirements) for details.

## Unified Workflow

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║                           DOCUMENT INTELLIGENCE                                ║
║                              Complete Workflow                                 ║
╚═══════════════════════════════════════════════════════════════════════════════╝


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STEP 1: FILE LOCATION (WITH GOOGLE DRIVE AUTOMATION)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   Where is your file?

   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
   │  Snowflake  │   │    Local    │   │ Google Drive│   │  External   │   │   Cloud     │
   │    Stage    │   │    File     │   │   Folder    │   │    Stage    │   │  Storage    │
   │             │   │             │   │  (CoCo)     │   │             │   │  Connector  │
   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
          │                 │                 │                 │                 │
          │                 ▼                 ▼                 │                 ▼
          │          ┌─────────────┐   ┌─────────────┐          │          ┌─────────────┐
          │          │ PUT command │   │  Cmd+A →    │          │          │  openflow   │
          │          │ SnowSQL/CLI │   │  Download → │          │          │   skill     │
          │          │             │   │  Extract →  │          │          │             │
          │          └──────┬──────┘   │  PUT Upload │          │          └──────┬──────┘
          │                 │          └──────┬──────┘          │                 │
          │                 │                 │                 │                 │
          └─────────────────┴─────────────────┴─────────────────┴─────────────────┘
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │ Files in Snowflake  │
                                   │       Stage         │
                                   └─────────────────────┘

   NEW: Google Drive Bulk Download (CoCo Desktop Only)
   • Detects if Google Drive folder is already open in browser
   • If not, opens provided Google Drive URL
   • Automates: Cmd+A → Right-click → Download
   • Waits for ZIP download and extraction
   • Uploads all files to Snowflake stage via PUT
   • Returns list of staged file paths


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STEP 2: EXTRACTION GOAL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   What do you want to extract?

   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
   │    Structured   │   │     Content     │   │     Visual      │
   │  Fields/Tables  │   │    Parsing +    │   │    Analysis     │
   │                 │   │     Images      │   │                 │
   │  (invoices,     │   │                 │   │  (charts,       │
   │   contracts,    │   │  (full text,    │   │   diagrams,     │
   │   forms)        │   │   RAG prep,     │   │   blueprints,   │
   │                 │   │   layout)       │   │   drawings)     │
   └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
            │                     │                     │
            ▼                     ▼                     ▼
      ┌───────────┐         ┌───────────┐         ┌───────────┐
      │  FLOW A   │         │  FLOW B   │         │  FLOW C   │
      │AI_EXTRACT │         │AI_PARSE_  │         │AI_COMPLETE│
      │           │         │ DOCUMENT  │         │  (Vision) │
      └───────────┘         └───────────┘         └───────────┘


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STEP 3: EXECUTION FLOWS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────────────────────┐
│ FLOW A: Structured Field Extraction (AI_EXTRACT)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Input: Invoices, Contracts, Forms, Receipts, Applications                 │
│   Output: JSON with extracted fields and tables                             │
│                                                                             │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│   │   Define     │──▶│   Build      │──▶│   Execute    │──▶│   Parse    │  │
│   │   Fields     │   │   Schema     │   │  AI_EXTRACT  │   │   JSON     │  │
│   └──────────────┘   └──────────────┘   └──────────────┘   └────────────┘  │
│                                                                             │
│   Constraints: 100MB max, 125 pages max, 100 entity questions/call          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ FLOW B: Content Parsing + Image Extraction (AI_PARSE_DOCUMENT)              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Input: Reports, Research Papers, Manuals, Policies, Books                 │
│   Output: Markdown with preserved layout, tables, headers                   │
│                                                                             │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│   │   Choose     │──▶│    Page      │──▶│   Execute    │──▶│  Process   │  │
│   │    Mode      │   │ Optimization │   │ AI_PARSE_DOC │   │  Markdown  │  │
│   │ LAYOUT / OCR │   │  (filter)    │   │              │   │            │  │
│   └──────────────┘   └──────────────┘   └──────────────┘   └────────────┘  │
│                                                                             │
│   Page Optimization Options:                                                │
│   • Parse entire document (all pages)                                       │
│   • Parse specific page range (e.g., pages 1-50)                           │
│   • Parse specific pages (e.g., pages 1, 5, 10, 25)                        │
│                                                                             │
│   Constraints: 50MB max, 500 pages max (use page_filter for larger)         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ FLOW C: Visual Analysis (AI_COMPLETE - Direct Vision)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Input: Charts, Graphs, Diagrams, Engineering Drawings, Blueprints,        │
│          Floor Plans, Flowcharts, Infographics, Technical Schematics        │
│   Output: Structured analysis, extracted data points, descriptions          │
│                                                                             │
│   IMPORTANT: AI_COMPLETE requires IMAGE files (PNG, JPEG, etc.)             │
│   If source is PDF → Convert to images first using pdf2image                │
│                                                                             │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│   │ Check File   │──▶│  Convert to  │──▶│    Send to   │──▶│   Get AI   │  │
│   │   Format     │   │   Image (if  │   │  AI_COMPLETE │   │  Analysis  │  │
│   │  (PDF/Image) │   │    PDF)      │   │   (Vision)   │   │            │  │
│   └──────────────┘   └──────────────┘   └──────────────┘   └────────────┘  │
│                                                                             │
│   PDF to Image Conversion (if needed):                                      │
│   • Stored procedure: convert_pdf_to_images                                 │
│   • Uses pdf2image package (Pillow)                                         │
│   • Configurable DPI, format, specific pages                                │
│   • Outputs to images_stage                                                 │
│                                                                             │
│   Analysis Prompts by Content Type:                                         │
│   • Charts/Graphs:   "Extract chart type, axes, data points, trends"        │
│   • Blueprints:      "Identify components, dimensions, scale, labels"       │
│   • Flowcharts:      "Describe process flow, nodes, connections"            │
│   • Eng. Drawings:   "Extract specifications, tolerances, materials"        │
│   • Floor Plans:     "Identify rooms, dimensions, features, layout"         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STEP 4: TEST RUN & REFINEMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   For structured extraction (Flow A), iteratively refine prompts:

   ┌─────────────────────────────────────────────────────────────────────┐
   │                                                                     │
   │   Define Fields ──▶ Select Test File ──▶ Run Single Extraction      │
   │                                                │                    │
   │                                                ▼                    │
   │                                         ┌───────────┐               │
   │                                         │ Satisfied?│               │
   │                                         └─────┬─────┘               │
   │                           ┌─────────NO───────┴───────YES──────┐     │
   │                           ▼                                   ▼     │
   │                    ┌─────────────┐                    ┌───────────┐ │
   │                    │  LLM Judge  │                    │  Choose   │ │
   │                    │   Refines   │                    │   Next    │ │
   │                    │   Prompt    │                    │  Action   │ │
   │                    └──────┬──────┘                    └───────────┘ │
   │                           │                                         │
   │                    Retry (max 3x)                                   │
   │                           │                                         │
   │                    Still failing?                                   │
   │                           │                                         │
   │                           ▼                                         │
   │                    ┌─────────────┐                                  │
   │                    │  FALLBACK   │                                  │
   │                    │ AI_PARSE +  │                                  │
   │                    │ AI_COMPLETE │                                  │
   │                    │ (structured │                                  │
   │                    │  outputs)   │                                  │
   │                    └─────────────┘                                  │
   │                                                                     │
   └─────────────────────────────────────────────────────────────────────┘

   Next Action Options:
   • Batch extraction (process all files)
   • Add more fields
   • Store result
   • Create pipeline
   • Done

   ┌─────────────────────────────────────────────────────────────────────┐
   │ FALLBACK: AI_PARSE_DOCUMENT + AI_COMPLETE Structured Outputs       │
   ├─────────────────────────────────────────────────────────────────────┤
   │                                                                     │
   │   When AI_EXTRACT fails after 3 refinement attempts:               │
   │                                                                     │
   │   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐           │
   │   │    Parse     │──▶│    Build     │──▶│  AI_COMPLETE │           │
   │   │   Document   │   │    JSON      │   │   with JSON  │           │
   │   │  (LAYOUT)    │   │   Schema     │   │    Schema    │           │
   │   └──────────────┘   └──────────────┘   └──────────────┘           │
   │                                                                     │
   │   Benefits:                                                         │
   │   • Better for complex nested data (addresses, line items)         │
   │   • Schema enforces data types (string, number, boolean)           │
   │   • Supports enums, arrays, nested objects                         │
   │   • More context from LAYOUT parsing                               │
   │                                                                     │
   │   Docs: https://docs.snowflake.com/en/user-guide/snowflake-cortex/ │
   │         complete-structured-outputs                                │
   │                                                                     │
   └─────────────────────────────────────────────────────────────────────┘


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STEP 5: POST-PROCESSING & PIPELINE OPTIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   What would you like to do with the results?

   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
   │    Done     │   │   Store     │   │   Create    │
   │  (one-time) │   │  Results    │   │  Pipeline   │
   └─────────────┘   └──────┬──────┘   └──────┬──────┘
                            │                 │
                            ▼                 ▼
                     ┌─────────────┐   ┌─────────────┐
                     │CREATE TABLE │   │  Stream +   │
                     │  + INSERT   │   │    Task     │
                     └─────────────┘   └─────────────┘

   ┌─────────────────────────────────────────────────────────────────────┐
   │ STORE RESULTS / PIPELINE CREATION - Optimization Questions        │
   ├─────────────────────────────────────────────────────────────────────┤
   │                                                                     │
   │   Question 1: "Can any files exceed the page limit?"               │
   │   • AI_EXTRACT: 125 pages max                                       │
   │   • AI_PARSE_DOCUMENT: 500 pages max                                │
   │                                                                     │
   │   Question 2: "Process entire document or specific pages?"         │
   │   (For AI_EXTRACT - reduces cost and improves speed)               │
   │                                                                     │
   │   ┌──────────────────────┐  ┌──────────────────┐  ┌─────────────┐  │
   │   │  Full Document       │  │  Specific Pages  │  │ Large Files │  │
   │   │                      │  │                  │  │  (>125 pg)  │  │
   │   │  Standard Pipeline   │  │  Page-Optimized  │  │  Chunking   │  │
   │   │  • All pages         │  │  • Extract pages │  │  Pipeline   │  │
   │   │  • Direct extraction │  │  • Then extract  │  │             │  │
   │   └──────────────────────┘  └──────────────────┘  └─────────────┘  │
   │                                                                     │
   │   Page Optimization Options:                                        │
   │   • First page only (invoices) → ~10% cost, ~10x faster            │
   │   • Page range (1-5)           → ~50% cost, ~2x faster             │
   │   • Specific pages (1,3,10)    → ~30% cost, ~3x faster             │
   │                                                                     │
   └─────────────────────────────────────────────────────────────────────┘
```

## Quick Start Examples

### Flow A: Extract Invoice Fields (AI_EXTRACT)
```sql
SELECT AI_EXTRACT(
  file => TO_FILE('@my_stage', 'invoice.pdf'),
  responseFormat => {
    'invoice_number': 'What is the invoice number?',
    'vendor': 'What is the vendor name?',
    'total': 'What is the total amount?'
  }
);
```

### Flow A: Page-Optimized Extraction (First Page Only)
For documents where all key info is on the first page (invoices, forms):
```sql
-- Step 1: Extract first page only (one-time setup: create extract_pdf_pages procedure)
CALL db.schema.extract_pdf_pages(
  '@my_stage', 'invoice.pdf', '@extracted_pages_stage', 1
);

-- Step 2: Run AI_EXTRACT on the single page (faster + cheaper)
SELECT AI_EXTRACT(
  file => TO_FILE('@extracted_pages_stage', 'invoice_page_1.pdf'),
  responseFormat => {
    'invoice_number': 'What is the invoice number?',
    'vendor': 'What is the vendor name?',
    'total': 'What is the total amount?'
  }
);
```

### Flow B: Parse Document for RAG (AI_PARSE_DOCUMENT)
```sql
-- With embeddings for vector search
INSERT INTO doc_chunks (content, embedding)
SELECT 
  f.value:content::STRING,
  SNOWFLAKE.CORTEX.EMBED_TEXT_1024('voyage-multilingual-2', f.value:content::STRING)
FROM (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@my_stage', 'document.pdf'),
    {'mode': 'LAYOUT', 'page_split': true}
  ) AS parsed
), LATERAL FLATTEN(input => parsed:pages) f;
```

### Flow B: Page Optimization Examples
```sql
-- Parse entire document
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'LAYOUT', 'page_split': true}
);

-- Parse specific page range (pages 1-50, 0-indexed)
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'LAYOUT', 'page_filter': [{'start': 0, 'end': 50}]}
);

-- Parse specific pages (pages 1, 5, 10, 25 → 0-indexed: 0, 4, 9, 24)
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'LAYOUT', 'page_filter': [0, 4, 9, 24]}
);

-- Parse multiple ranges (pages 1-10 and 50-60)
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'LAYOUT', 'page_filter': [{'start': 0, 'end': 10}, {'start': 49, 'end': 60}]}
);
```

### Flow C: Analyze Chart/Blueprint (AI_COMPLETE)

**If image file (PNG, JPEG, etc.) - analyze directly:**
```sql
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@my_stage', 'chart.png')}},
        {'type': 'text', 'text': 'Analyze this chart. Extract all data points, labels, and trends.'}
      ]
    }
  ],
  {'max_tokens': 4096}
);
```

**If PDF file - convert to image first:**
```sql
-- Step 1: Create conversion procedure and images stage (one-time setup)
-- See SKILL.md Step 2.6 for full procedure code

-- Step 2: Convert PDF to images
CALL db.schema.convert_pdf_to_images(
  '@my_stage',
  'blueprint.pdf',
  '@images_stage',
  200,    -- DPI
  'PNG',  -- Format
  [1]     -- Specific pages (or NULL for all)
);

-- Step 3: Analyze converted image
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@images_stage', 'blueprint_page_1.png')}},
        {'type': 'text', 'text': 'Analyze this blueprint. Identify all components, dimensions, and specifications.'}
      ]
    }
  ],
  {'max_tokens': 4096}
);
```

### Fallback: AI_PARSE_DOCUMENT + AI_COMPLETE Structured Outputs
When AI_EXTRACT doesn't produce accurate results, use this alternative:
```sql
-- Parse document and extract with structured JSON output
WITH parsed_doc AS (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@my_stage', 'invoice.pdf'),
    {'mode': 'LAYOUT'}
  ):content::STRING AS document_text
)
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  'Extract invoice information from this document:

' || document_text,
  {
    'response_format': {
      'type': 'json',
      'schema': {
        'type': 'object',
        'properties': {
          'invoice_number': {'type': 'string'},
          'vendor_name': {'type': 'string'},
          'total_amount': {'type': 'number'},
          'line_items': {
            'type': 'array',
            'items': {
              'type': 'object',
              'properties': {
                'description': {'type': 'string'},
                'amount': {'type': 'number'}
              }
            }
          }
        },
        'required': ['invoice_number', 'vendor_name', 'total_amount']
      }
    },
    'max_tokens': 4096
  }
) AS extraction
FROM parsed_doc;
```

## Large Documents (>125 pages for AI_EXTRACT, >500 pages for AI_PARSE_DOCUMENT)

| Function | Max Pages | Strategy |
|----------|-----------|----------|
| AI_PARSE_DOCUMENT | 500 | Use `page_filter` with ranges: `[{'start': 0, 'end': 500}]` |
| AI_EXTRACT | 125 | Option A: Chunking (125-page chunks) OR Option B: Full document (AI_PARSE + AI_COMPLETE) |

### Quick Page Count Check
```sql
-- Get total pages before processing
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'OCR', 'page_filter': [{'start': 0, 'end': 1}]}
):pageCount AS total_pages;
```

### Large File Handling Options

When files exceed 125 pages, users choose between two approaches:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ LARGE FILE OPTIONS (>125 pages for AI_EXTRACT)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌───────────────────────────────┐    ┌───────────────────────────────┐   │
│   │ OPTION A: Chunk into segments │    │ OPTION B: Keep doc together   │   │
│   │                               │    │                               │   │
│   │ • Split into 125-page chunks  │    │ • AI_PARSE_DOCUMENT (≤500pg)  │   │
│   │ • Process each with AI_EXTRACT│    │ • AI_COMPLETE + structured    │   │
│   │ • Aggregate results           │    │   outputs for extraction      │   │
│   │                               │    │                               │   │
│   │ Best for:                     │    │ Best for:                     │   │
│   │ • Invoices, receipts          │    │ • Contracts (cross-refs)      │   │
│   │ • Forms, applications         │    │ • Legal documents             │   │
│   │ • Independent pages           │    │ • Research papers             │   │
│   │                               │    │ • Insurance policies          │   │
│   │                               │    │ • Loan agreements             │   │
│   │                               │    │ • Technical manuals           │   │
│   └───────────────────────────────┘    └───────────────────────────────┘   │
│                                                                             │
│   Comparison:                                                               │
│   • Chunking: Unlimited pages, but loses cross-page context                │
│   • Full doc: Up to 500 pages, preserves all document context              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Full Document Extraction (Option B - for contracts, legal docs)
```sql
-- Parse entire document, then extract with structured output
WITH parsed_doc AS (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@my_stage', 'contract.pdf'),
    {'mode': 'LAYOUT'}
  ):content::STRING AS document_text
)
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  'Extract contract information from this document:

' || document_text,
  {
    'response_format': {
      'type': 'json',
      'schema': {
        'type': 'object',
        'properties': {
          'contract_parties': {
            'type': 'object',
            'properties': {
              'party_a': {'type': 'string'},
              'party_b': {'type': 'string'}
            }
          },
          'effective_date': {'type': 'string'},
          'total_value': {'type': 'string'},
          'key_terms': {'type': 'array', 'items': {'type': 'string'}},
          'governing_law': {'type': 'string'}
        },
        'required': ['contract_parties', 'effective_date', 'total_value']
      }
    },
    'max_tokens': 8192
  }
) AS contract_data
FROM parsed_doc;
```

### Pipeline Chunking (Option A - for independent pages)

When creating a pipeline, the skill asks: **"Can any files be longer than 125 pages?"**

**If YES**, it sets up a chunking pipeline:

```
┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│   New File     │───▶│  Check Pages   │───▶│  >125 pages?   │
│   Detected     │    │   (quick OCR)  │    │                │
└────────────────┘    └────────────────┘    └───────┬────────┘
                                                    │
                           ┌────────────────────────┴─────────────────────┐
                           │ YES                                      NO  │
                           ▼                                          ▼   │
                    ┌────────────────┐                      ┌────────────────┐
                    │ Split into     │                      │    Process     │
                    │ 125-page chunks│                      │    Directly    │
                    │ (stored proc)  │                      │                │
                    └───────┬────────┘                      └───────┬────────┘
                            │                                       │
                            ▼                                       │
                    ┌────────────────┐                              │
                    │ Track chunks   │                              │
                    │ in metadata    │                              │
                    │ table          │                              │
                    └───────┬────────┘                              │
                            │                                       │
                            └───────────────────┬───────────────────┘
                                                │
                                                ▼
                                    ┌────────────────────┐
                                    │  Extract/Parse     │
                                    │  (AI_EXTRACT or    │
                                    │   AI_PARSE_DOC)    │
                                    └─────────┬──────────┘
                                              │
                                              ▼
                                    ┌────────────────────┐
                                    │  Aggregate Results │
                                    │  (join chunks)     │
                                    └────────────────────┘
```

**Pipeline components created:**

| Component | Purpose |
|-----------|---------|
| `split_document_into_chunks` | Stored procedure (PyPDF2) - splits PDFs into 125-page chunks |
| `document_chunks` | Tracking table - records source file, chunk files, page ranges |
| `chunk_documents_task` | Task 1 - detects new files, checks page count, chunks if needed |
| `extract_from_chunks_task` | Task 2 - extracts from chunks, stores results |
| `extraction_results_aggregated` | View - joins chunk results back to source document |

```sql
-- Check processing status
SELECT 
  source_file,
  total_chunks,
  invoice_number,
  vendor_name,
  total_amount
FROM db.schema.extraction_results_aggregated;
```

## Local File Upload

```bash
# SnowSQL
PUT file:///path/to/file.pdf @my_stage AUTO_COMPRESS=FALSE;

# Snowflake CLI  
snow stage copy /path/to/file.pdf @my_stage --overwrite

# Python
from snowflake.snowpark import Session
session.file.put('/path/to/file.pdf', '@my_stage', auto_compress=False)
```

## Prerequisites

- Snowflake account with Cortex AI access
- `SNOWFLAKE.CORTEX_USER` database role granted
- Files in Snowflake stage

## Google Drive Automation (CoCo Desktop)

This edition includes automated bulk download from Google Drive folders:

### Prerequisites
- CoCo Desktop app with agentic browser capability
- User authenticated to Google Drive
- Snowflake stage configured for file uploads

### Workflow
```
1. User has Google Drive folder open (or provides URL)
   ↓
2. Skill detects CoCo Desktop environment and browser state
   ↓
3. Automated steps:
   • Cmd+A (select all files)
   • Right-click → Download
   • Wait for ZIP creation and download
   • Extract ZIP to local folder
   • Upload all files to Snowflake stage via PUT
   ↓
4. Files ready for extraction/parsing workflows
```

### Supported Platforms
- **macOS**: Cmd+A, ~/Downloads/
- **Windows**: Ctrl+A, %USERPROFILE%\Downloads\
- **Linux**: Ctrl+A, ~/Downloads/

### Error Handling
- Authentication failures → Prompts user to log in manually
- Download failures → Fallbacks to openflow or manual upload
- Large folders → Shows progress, handles ZIP creation delays

## Installation

Install using Cortex Code CLI:

```bash
cortex skill add https://github.com/sfc-gh-nejain/document-intelligence-coco-desktop
```

Invoke with: `/document-intelligence-coco-desktop`

## License

MIT License
