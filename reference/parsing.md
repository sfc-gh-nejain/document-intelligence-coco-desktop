# Parsing Sub-Skill

Full content parsing using AI_PARSE_DOCUMENT with page optimization.

## When to Use

This sub-skill is triggered when the user selects:
- "Full text with layout preservation (tables, headers, structure)"
- "Text only via OCR (quick text extraction)"

## AI_PARSE_DOCUMENT Constraints

**Inform the user before proceeding:**

| Constraint | Limit |
|------------|-------|
| Max file size | 50 MB |
| Max pages | 500 per call |
| Output format | Markdown |

## Supported Formats

PDF, PNG, PPTX/PPT, DOC/DOCX, JPEG/JPG, HTM/HTML, TEXT/TXT, TIF/TIFF, BMP, GIF, WEBP

**Note:** CSV, MD, and EML files are NOT supported by AI_PARSE_DOCUMENT (use AI_EXTRACT instead).

## Parsing Modes

| Mode | Best For | Output |
|------|----------|--------|
| LAYOUT | Documents with tables, headers, structure | Markdown with formatting |
| OCR | Scanned documents, images of text | Plain text |

---

## Step 1: Page Optimization

Before parsing, ask user about page optimization to improve performance and reduce costs.

**Ask** user:
```
Would you like to parse the entire document or specific pages?
Options:
1. Parse entire document
2. Parse specific page range (e.g., pages 1-50)
3. Parse specific pages (e.g., pages 1, 5, 10, 25)
4. I'm not sure - help me decide
```

---

### If "Parse entire document"

```sql
-- Parse all pages with page_split for easier processing
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'LAYOUT', 'page_split': true}
) AS parsed;
```

---

### If "Parse specific page range"

Ask for the range:
```
What page range would you like to parse?
Example: 1-50 (parses pages 1 through 50)
Note: Page numbers are 1-indexed for your input, converted to 0-indexed internally.
```

```sql
-- Parse specific range (page_filter uses 0-indexed pages)
-- For user input "1-50", use start: 0, end: 50
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'LAYOUT', 'page_filter': [{'start': 0, 'end': 50}]}
) AS parsed;
```

---

### If "Parse specific pages"

Ask for page numbers:
```
Which specific pages would you like to parse?
Example: 1, 5, 10, 25 (parses only these pages)
Note: Page numbers are 1-indexed for your input, converted to 0-indexed internally.
```

```sql
-- Parse specific pages (0-indexed internally)
-- For user input "1, 5, 10, 25", use [0, 4, 9, 24]
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'LAYOUT', 'page_filter': [0, 4, 9, 24]}
) AS parsed;
```

---

### If "I'm not sure"

Provide guidance:
```
Here's how to decide:

- Parse entire document: Best for full text extraction, RAG pipelines, 
  or when you need all content. Works well for documents ≤500 pages.

- Parse specific page range: Best when you know the relevant section 
  (e.g., "chapters 1-3 are on pages 1-75"). Faster and cheaper.

- Parse specific pages: Best for extracting just a table of contents, 
  specific chapters, or known important pages.

Tips:
- First page often has title, author, date
- Last few pages often have references, appendix
- Table of contents usually in first 5 pages

Would you like me to first get the page count so you can decide?
```

If user wants page count:
```sql
-- Quick page count check (minimal processing)
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@stage', 'document.pdf'),
  {'mode': 'OCR', 'page_filter': [{'start': 0, 'end': 1}]}
):pageCount AS total_pages;
```

---

## Step 2: Execute Parsing

### LAYOUT Mode (preserves structure)

```sql
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@db.schema.stage', 'document.pdf'),
  {'mode': 'LAYOUT', 'page_split': true}
) AS parsed_result;
```

### OCR Mode (text extraction)

```sql
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@db.schema.stage', 'document.pdf'),
  {'mode': 'OCR'}
) AS parsed_result;
```

---

## Step 3: Working with Results

### Extract content from parsed result

```sql
SELECT 
  parsed_result:content::STRING AS full_text,
  parsed_result:pageCount::INT AS total_pages
FROM (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@db.schema.stage', 'document.pdf'),
    {'mode': 'LAYOUT'}
  ) AS parsed_result
);
```

### Extract individual pages (when page_split is true)

```sql
SELECT 
  page.value:index::INT AS page_num,
  page.value:content::STRING AS page_content
FROM (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@db.schema.stage', 'document.pdf'),
    {'mode': 'LAYOUT', 'page_split': true}
  ) AS parsed_result
),
LATERAL FLATTEN(input => parsed_result:pages) page;
```

---

## Step 4: Batch Processing

### Process multiple documents

```sql
-- Enable directory table
ALTER STAGE @db.schema.stage SET DIRECTORY = (ENABLE = TRUE);
ALTER STAGE @db.schema.stage REFRESH;

-- Parse all PDFs in stage
SELECT 
  relative_path AS file_path,
  AI_PARSE_DOCUMENT(
    TO_FILE('@db.schema.stage', relative_path),
    {'mode': 'LAYOUT'}
  ):content::STRING AS parsed_content
FROM DIRECTORY(@db.schema.stage)
WHERE relative_path LIKE '%.pdf';
```

### Store parsed results

```sql
-- Create table for parsed documents
CREATE TABLE IF NOT EXISTS db.schema.parsed_documents (
  doc_id INT AUTOINCREMENT,
  file_path STRING,
  file_name STRING,
  total_pages INT,
  content TEXT,
  parsed_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Insert parsed content
INSERT INTO db.schema.parsed_documents (file_path, file_name, total_pages, content)
SELECT 
  relative_path,
  SPLIT_PART(relative_path, '/', -1),
  parsed_result:pageCount::INT,
  parsed_result:content::STRING
FROM (
  SELECT 
    relative_path,
    AI_PARSE_DOCUMENT(
      TO_FILE('@db.schema.stage', relative_path),
      {'mode': 'LAYOUT'}
    ) AS parsed_result
  FROM DIRECTORY(@db.schema.stage)
  WHERE relative_path LIKE '%.pdf'
);
```

---

## Large Document Handling (>500 pages)

For documents exceeding 500 pages, parse in chunks:

```sql
-- Parse document in chunks
CREATE OR REPLACE TEMPORARY TABLE doc_chunks AS
SELECT 
  page.value:index::INT AS page_num,
  page.value:content::STRING AS content
FROM (
  SELECT PARSE_JSON(AI_PARSE_DOCUMENT(
    TO_FILE('@stage', 'large_doc.pdf'),
    {'mode': 'LAYOUT', 'page_filter': [{'start': 0, 'end': 500}], 'page_split': true}
  )) AS parsed
  UNION ALL
  SELECT PARSE_JSON(AI_PARSE_DOCUMENT(
    TO_FILE('@stage', 'large_doc.pdf'),
    {'mode': 'LAYOUT', 'page_filter': [{'start': 500, 'end': 1000}], 'page_split': true}
  )) AS parsed
) chunks,
LATERAL FLATTEN(input => chunks.parsed:pages) page;
```

---

## Common Use Cases

### RAG Pipeline Preparation

```sql
-- Parse and chunk for RAG
WITH parsed AS (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@stage', 'manual.pdf'),
    {'mode': 'LAYOUT', 'page_split': true}
  ) AS result
)
SELECT 
  page.value:index::INT AS chunk_id,
  page.value:content::STRING AS chunk_text,
  LENGTH(page.value:content::STRING) AS chunk_length
FROM parsed,
LATERAL FLATTEN(input => result:pages) page;
```

### Full-Text Search Preparation

```sql
-- Extract searchable text
INSERT INTO db.schema.document_search_index (doc_id, file_name, searchable_text)
SELECT 
  doc_id,
  file_name,
  LOWER(content) AS searchable_text
FROM db.schema.parsed_documents;
```

---

## Supported Languages

Arabic, Bengali, Burmese, Cebuano, Chinese, Czech, Dutch, English, French, German, Hebrew, Hindi, Indonesian, Italian, Japanese, Khmer, Korean, Lao, Malay, Persian, Polish, Portuguese, Russian, Spanish, Tagalog, Thai, Turkish, Urdu, Vietnamese

---

## Next Steps: Post-Processing

After parsing is complete, **ask the user** what they want to do next:

→ **Load `reference/pipeline.md`** for post-processing options:
- One-time parsing (done)
- Store results in a Snowflake table
- Set up a continuous processing pipeline with streams and tasks
- Build RAG/search integration with vector embeddings

The pipeline sub-skill contains templates for parsing pipelines and page-level chunking patterns.

For detailed AI_PARSE_DOCUMENT function options, see `reference/ai-parse-doc-and-ai-complete.md`.
