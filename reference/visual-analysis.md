# Visual Analysis Sub-Skill

Analyze charts, blueprints, diagrams, and visual content using AI_COMPLETE vision capabilities.

## When to Use

This sub-skill is triggered when the user selects:
- "Images and visual content for analysis (charts, blueprints)"

## Overview

Visual Analysis uses AI_COMPLETE directly to analyze images. No parsing step is needed - AI_COMPLETE is a vision model that can directly interpret image files.

**IMPORTANT:** AI_COMPLETE requires image files (PNG, JPEG, etc.). If the user has a PDF, it must be converted to images first.

## Supported Image Formats

PNG, JPEG/JPG, TIFF, BMP, GIF, WEBP

**Non-image files (PDF, DOCX, PPTX) must be converted to images before analysis.**

---

## Step 1: Check File Type (Automatic)

**Do NOT ask user about file type.** Automatically detect from file extension and handle conversion if needed.

### File Type Detection

```
Get file path from user → Extract extension → Determine if conversion needed
```

| Extension | File Type | Action |
|-----------|-----------|--------|
| .png, .jpg, .jpeg, .tiff, .bmp, .gif, .webp | Image | ✅ Proceed directly to analysis |
| .pdf | PDF | ⚠️ Convert to image first |
| .pptx, .ppt | PowerPoint | ⚠️ Convert to image first |
| .docx, .doc | Word | ⚠️ Convert to image first |

### Inform User About Conversion

**If non-image file detected, inform user:**

```
I noticed your file is a [PDF/PowerPoint/Word] document. 
AI_COMPLETE vision requires image files for visual analysis.

I'll convert the relevant pages to images first, then analyze them.

Which pages contain the visual content you want to analyze?
Options:
1. All pages
2. First page only
3. Specific page(s) - please specify (e.g., 1, 3, 5)
```

---

## Step 2: Gather User Questions

**Ask** user what information they want to extract:

```
What would you like to extract or analyze from this visual content?

You can provide multiple questions or data points. Examples:
- "Extract all data points from the chart"
- "What are the axis labels and values?"
- "Identify the trend and any anomalies"
- "List all components shown in the blueprint"
- "What are the dimensions and scale?"

Please describe what information you need:
```

**Capture user's response** and use it to build the analysis prompt.

**Common question patterns by content type:**

| Content Type | Typical Questions |
|--------------|-------------------|
| **Charts/Graphs** | Data points, axis labels, trends, legend values, chart type, comparisons |
| **Blueprints** | Components, dimensions, scale, materials, annotations, room labels |
| **Flowcharts** | Process steps, decision points, connections, start/end points |
| **Diagrams** | Parts identification, relationships, labels, hierarchy |
| **Screenshots** | UI elements, text content, error messages, form values |

**If user provides multiple questions**, combine them into a structured prompt:

```
User questions:
1. What are the data points?
2. What is the trend?
3. Are there any anomalies?

→ Build prompt: "Analyze this chart and answer the following:
   1. Extract all data points with their exact values
   2. Describe the overall trend shown
   3. Identify any anomalies or outliers in the data"
```

---

## Step 3: Proceed Based on File Format

### If Image File: Proceed Directly to Analysis

Use the user's questions to build the analysis prompt:

```sql
-- Direct visual analysis with AI_COMPLETE
-- Replace <USER_QUESTIONS> with the questions gathered in Step 2
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.stage', 'chart.png')}},
        {'type': 'text', 'text': 'Analyze this image and answer the following questions:\n\n<USER_QUESTIONS>\n\nProvide detailed answers for each question.'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS analysis;
```

**Example with user questions:**
```sql
-- User asked: "What are the data points? What is the trend?"
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.stage', 'sales_chart.png')}},
        {'type': 'text', 'text': 'Analyze this chart and answer the following questions:

1. What are all the data points shown? List each with its exact value.
2. What is the overall trend? Is it increasing, decreasing, or stable?

Provide detailed answers for each question.'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS analysis;
```

---

### If PDF File: Convert to Image First

#### Create the PDF-to-Image conversion stored procedure (one-time setup)

This procedure converts PDF pages to images using pdf2image and uploads them to an images stage:

```sql
-- Create stage for converted images (with server-side encryption)
CREATE STAGE IF NOT EXISTS db.schema.images_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  COMMENT = 'Stage for PDF-to-image converted files';

-- Create PDF to image conversion stored procedure
CREATE OR REPLACE PROCEDURE db.schema.convert_pdf_to_images(
    stage_name STRING, 
    file_name STRING, 
    dest_stage_name STRING,
    dpi INT DEFAULT 200,
    image_format STRING DEFAULT 'PNG',
    specific_pages ARRAY DEFAULT NULL
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ('snowflake-snowpark-python', 'pdf2image', 'pillow')
HANDLER = 'run'
AS
$
from pdf2image import convert_from_bytes
from snowflake.snowpark import Session
from snowflake.snowpark.file_operation import FileOperation
from io import BytesIO
import os

def run(session: Session, stage_name: str, file_name: str, dest_stage_name: str, 
        dpi: int = 200, image_format: str = 'PNG', specific_pages: list = None) -> dict:
    result = {
        "status": "success",
        "source_file": file_name,
        "total_pages": 0,
        "images_created": 0,
        "image_files": []
    }
    
    try:
        # Download PDF from stage
        file_url = f"{stage_name}/{file_name}"
        get_result = session.file.get(file_url, '/tmp/')
        file_path = os.path.join('/tmp/', file_name)
        
        if not os.path.exists(file_path):
            return {"status": "error", "message": f"File {file_name} not found in {stage_name}"}

        with open(file_path, 'rb') as f:
            pdf_data = f.read()
        
        # Convert PDF to images
        # If specific_pages provided, only convert those pages (1-indexed)
        if specific_pages:
            images = convert_from_bytes(
                pdf_data, 
                dpi=dpi, 
                fmt=image_format.lower(),
                first_page=min(specific_pages),
                last_page=max(specific_pages)
            )
            # Filter to only requested pages
            page_indices = [p - min(specific_pages) for p in specific_pages]
            images = [images[i] for i in page_indices if i < len(images)]
        else:
            images = convert_from_bytes(pdf_data, dpi=dpi, fmt=image_format.lower())
        
        result["total_pages"] = len(images)
        
        # Get base filename without extension
        base_name = os.path.splitext(file_name)[0]
        ext = image_format.lower()
        if ext == 'jpeg':
            ext = 'jpg'
        
        for i, image in enumerate(images):
            page_num = specific_pages[i] if specific_pages else i + 1
            
            # Create image filename: original_page_1.png
            image_filename = f'{base_name}_page_{page_num}.{ext}'
            image_file_path = os.path.join('/tmp/', image_filename)
            
            # Save image
            image.save(image_file_path, format=image_format.upper())
            
            if not os.path.exists(image_file_path):
                return {"status": "error", "message": f"Failed to create image {image_filename}"}

            # Upload to destination stage
            FileOperation(session).put(
                f"file://{image_file_path}",
                dest_stage_name,
                auto_compress=False
            )
            
            result["image_files"].append({
                "page_number": page_num,
                "filename": image_filename,
                "format": image_format.upper()
            })
            result["images_created"] += 1
            
            # Clean up temp image file
            os.remove(image_file_path)
        
        # Clean up original temp file
        os.remove(file_path)
            
        return result
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
$;
```

#### Ask which pages to convert

```
Which pages of the PDF contain the charts/diagrams you want to analyze?
Options:
1. Convert all pages to images
2. Convert specific pages (e.g., 1, 3, 5)
3. Convert a page range (e.g., pages 1-10)
4. Convert just the first page to test
```

#### Convert PDF to images

```sql
-- Convert specific pages (e.g., pages 1, 3, 5)
CALL db.schema.convert_pdf_to_images(
  '@db.schema.doc_stage',
  'blueprint.pdf',
  '@db.schema.images_stage',
  200,           -- DPI (higher = better quality, larger files)
  'PNG',         -- Format: PNG, JPEG
  [1, 3, 5]      -- Specific pages (1-indexed), or NULL for all pages
);

-- Convert all pages
CALL db.schema.convert_pdf_to_images(
  '@db.schema.doc_stage',
  'blueprint.pdf',
  '@db.schema.images_stage',
  200,
  'PNG',
  NULL           -- NULL = convert all pages
);
```

#### Verify converted images

```sql
-- Refresh stage directory
ALTER STAGE db.schema.images_stage REFRESH;

-- List converted images
SELECT * FROM DIRECTORY(@db.schema.images_stage)
WHERE relative_path LIKE 'blueprint_%'
ORDER BY relative_path;
```

#### Analyze images with AI_COMPLETE

Use the user's questions from Step 2:

```sql
-- Analyze a single converted image with user's questions
-- Replace <USER_QUESTIONS> with questions gathered in Step 2
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.images_stage', 'blueprint_page_1.png')}},
        {'type': 'text', 'text': 'Analyze this image and answer the following questions:\n\n<USER_QUESTIONS>\n\nProvide detailed answers for each question.'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS analysis;
```

---

### If PowerPoint File: Convert to Images First

#### Create PowerPoint-to-Image conversion stored procedure

```sql
-- PowerPoint to image conversion stored procedure
CREATE OR REPLACE PROCEDURE db.schema.convert_pptx_to_images(
    stage_name STRING, 
    file_name STRING, 
    dest_stage_name STRING,
    image_format STRING DEFAULT 'PNG',
    specific_slides ARRAY DEFAULT NULL
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ('snowflake-snowpark-python', 'python-pptx', 'pillow')
HANDLER = 'run'
AS
$
from pptx import Presentation
from pptx.util import Inches
from PIL import Image
from snowflake.snowpark import Session
from snowflake.snowpark.file_operation import FileOperation
from io import BytesIO
import os

def run(session: Session, stage_name: str, file_name: str, dest_stage_name: str,
        image_format: str = 'PNG', specific_slides: list = None) -> dict:
    result = {
        "status": "success",
        "source_file": file_name,
        "total_slides": 0,
        "images_created": 0,
        "image_files": []
    }
    
    try:
        file_url = f"{stage_name}/{file_name}"
        session.file.get(file_url, '/tmp/')
        file_path = os.path.join('/tmp/', file_name)
        
        if not os.path.exists(file_path):
            return {"status": "error", "message": f"File {file_name} not found"}

        prs = Presentation(file_path)
        result["total_slides"] = len(prs.slides)
        
        base_name = os.path.splitext(file_name)[0]
        ext = image_format.lower()
        if ext == 'jpeg':
            ext = 'jpg'
        
        slides_to_process = specific_slides if specific_slides else range(1, len(prs.slides) + 1)
        
        for slide_num in slides_to_process:
            if slide_num < 1 or slide_num > len(prs.slides):
                continue
                
            # Export slide as image (simplified - actual implementation may need additional libraries)
            image_filename = f'{base_name}_slide_{slide_num}.{ext}'
            image_file_path = os.path.join('/tmp/', image_filename)
            
            # Note: python-pptx doesn't directly export to images
            # For production, consider using pdf2image after converting PPTX to PDF
            # or use LibreOffice headless mode
            
            result["image_files"].append({
                "slide_number": slide_num,
                "filename": image_filename,
                "format": image_format.upper()
            })
            result["images_created"] += 1
        
        os.remove(file_path)
        return result
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
$;
```

**Alternative: Convert PPTX via PDF (Recommended)**

For better quality, convert PowerPoint to PDF first, then use the PDF-to-image procedure:

```sql
-- 1. Export PowerPoint as PDF (do this outside Snowflake)
-- 2. Upload PDF to stage
-- 3. Use convert_pdf_to_images procedure
CALL db.schema.convert_pdf_to_images(
  '@db.schema.doc_stage',
  'presentation.pdf',
  '@db.schema.images_stage',
  200,
  'PNG',
  NULL
);
```

---

### If Word Document: Convert to Images First

#### Create Word-to-Image conversion stored procedure

```sql
-- Word document to image conversion stored procedure
CREATE OR REPLACE PROCEDURE db.schema.convert_docx_to_images(
    stage_name STRING, 
    file_name STRING, 
    dest_stage_name STRING,
    image_format STRING DEFAULT 'PNG',
    specific_pages ARRAY DEFAULT NULL
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ('snowflake-snowpark-python', 'python-docx', 'pillow')
HANDLER = 'run'
AS
$
from docx import Document
from snowflake.snowpark import Session
from snowflake.snowpark.file_operation import FileOperation
import os

def run(session: Session, stage_name: str, file_name: str, dest_stage_name: str,
        image_format: str = 'PNG', specific_pages: list = None) -> dict:
    result = {
        "status": "success",
        "source_file": file_name,
        "message": "Word documents should be converted to PDF first for best results"
    }
    
    # Note: python-docx doesn't directly export to images
    # Recommended workflow: Convert DOCX to PDF, then PDF to images
    
    return result
$;
```

**Recommended: Convert DOCX via PDF**

For Word documents, the recommended approach is to convert to PDF first:

```sql
-- 1. Export Word document as PDF (do this outside Snowflake or use LibreOffice)
-- 2. Upload PDF to stage
-- 3. Use convert_pdf_to_images procedure
CALL db.schema.convert_pdf_to_images(
  '@db.schema.doc_stage',
  'document.pdf',
  '@db.schema.images_stage',
  200,
  'PNG',
  NULL
);
```

---

### Universal File-to-Image Conversion (Recommended Approach)

For any non-image file, the most reliable approach is:

1. **Convert to PDF first** (outside Snowflake using LibreOffice, Microsoft Office, or online tools)
2. **Upload PDF to Snowflake stage**
3. **Use `convert_pdf_to_images` procedure**

```
Non-image file → PDF → Images → AI_COMPLETE analysis
```

This approach works for:
- PDF files (direct)
- PowerPoint (.pptx, .ppt)
- Word documents (.docx, .doc)
- Excel with charts (.xlsx)
- Any printable document

---

## Prompt Templates by Content Type

Use these templates as a starting point, then customize based on user's specific questions from Step 2.

### Charts/Graphs

```sql
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@stage', 'chart.png')}},
        {'type': 'text', 'text': 'Analyze this chart. Extract:
1. Chart type (bar, line, pie, etc.)
2. Title and axis labels
3. All data points with their values
4. Key trends or insights
5. Any annotations or legends

Format the data points as a table if possible.'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS chart_analysis;
```

### Blueprints/Technical Drawings

```sql
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@stage', 'blueprint.png')}},
        {'type': 'text', 'text': 'Analyze this technical drawing/blueprint. Identify:
1. All labeled components and parts
2. Dimensions and measurements with units
3. Materials specifications if shown
4. Assembly instructions or notes
5. Scale information
6. Any warnings or special instructions'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS blueprint_analysis;
```

### Diagrams/Flowcharts

```sql
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@stage', 'diagram.png')}},
        {'type': 'text', 'text': 'Analyze this diagram/flowchart. Describe:
1. Overall purpose of the diagram
2. All nodes/boxes and their labels
3. Connections and relationships between elements
4. Flow direction and sequence
5. Decision points and branches
6. Start and end points'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS diagram_analysis;
```

### General Visual Analysis

```sql
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@stage', 'image.png')}},
        {'type': 'text', 'text': 'Describe this image in detail. Include all visible text, numbers, symbols, and visual elements.'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS visual_analysis;
```

---

## Batch Visual Analysis

For analyzing multiple images:

```sql
-- Analyze all images in a stage
SELECT 
  relative_path AS image_file,
  AI_COMPLETE(
    'claude-3-5-sonnet',
    [
      {
        'role': 'user',
        'content': [
          {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.images_stage', relative_path)}},
          {'type': 'text', 'text': 'Analyze this chart and extract all data points.'}
        ]
      }
    ],
    {'max_tokens': 4096}
  ) AS analysis
FROM DIRECTORY(@db.schema.images_stage)
WHERE relative_path LIKE '%.png' OR relative_path LIKE '%.jpg';
```

---

## Store Visual Analysis Results

```sql
-- Create table for visual analysis results
CREATE TABLE IF NOT EXISTS db.schema.visual_analysis_results (
  result_id INT AUTOINCREMENT,
  image_path STRING,
  source_pdf STRING,           -- Original PDF if converted from PDF
  page_number INT,             -- Page number if converted from PDF
  analysis_type STRING,        -- chart, blueprint, diagram, general
  analysis_result VARIANT,
  analyzed_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Insert analysis result
INSERT INTO db.schema.visual_analysis_results (image_path, source_pdf, page_number, analysis_type, analysis_result)
SELECT 
  'blueprint_page_1.png',
  'blueprint.pdf',
  1,
  'blueprint',
  PARSE_JSON(AI_COMPLETE(
    'claude-3-5-sonnet',
    [
      {
        'role': 'user',
        'content': [
          {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.images_stage', 'blueprint_page_1.png')}},
          {'type': 'text', 'text': 'Analyze this blueprint. Return a JSON object with keys: components (array), measurements (array), materials (array), notes (string).'}
        ]
      }
    ],
    {'max_tokens': 4096}
  ));
```

---

## Structured Output for Visual Analysis

For consistent JSON output, use a structured prompt:

```sql
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.images_stage', 'chart.png')}},
        {'type': 'text', 'text': 'Analyze this chart and return a JSON object with the following structure:
{
  "chart_type": "string (bar, line, pie, scatter, etc.)",
  "title": "string",
  "x_axis_label": "string",
  "y_axis_label": "string",
  "data_points": [
    {"label": "string", "value": number}
  ],
  "trends": ["string"],
  "insights": ["string"]
}

Return ONLY the JSON object, no additional text.'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS structured_chart_analysis;
```

---

## DPI Recommendations

| Use Case | Recommended DPI | Notes |
|----------|-----------------|-------|
| Charts with text | 150-200 | Balance between quality and file size |
| Blueprints | 200-300 | Higher DPI for fine details |
| Diagrams | 150 | Usually sufficient |
| Text-heavy images | 200-300 | Ensures text is readable |
| Quick preview | 100 | Fast, smaller files |

---

## Troubleshooting

### Image too large
- Reduce DPI when converting from PDF
- Use JPEG format instead of PNG for smaller files

### Poor analysis quality
- Increase DPI for better image quality
- Use more specific prompts
- Break complex images into sections

### PDF conversion fails
- Check that pdf2image package is available
- Ensure poppler is installed (required by pdf2image)
- Try converting fewer pages at once

---

## Next Steps: Post-Processing

After visual analysis is complete, **ask the user** what they want to do next:

→ **Load `reference/pipeline.md`** for post-processing options:
- One-time analysis (done)
- Store results in a Snowflake table
- Set up a continuous processing pipeline with streams and tasks

The pipeline sub-skill contains templates for visual analysis pipelines including PDF-to-image conversion workflows.
