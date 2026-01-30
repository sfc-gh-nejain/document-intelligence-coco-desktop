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

---

## Step 1: Check File Format

**Ask** user:
```
What format is your file?
Options:
1. Image file (PNG, JPEG, TIFF, BMP, GIF, WEBP)
2. PDF file (needs conversion to image)
```

---

## If Image File: Proceed Directly to Analysis

```sql
-- Direct visual analysis with AI_COMPLETE
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.stage', 'chart.png')}},
        {'type': 'text', 'text': 'Analyze this chart/diagram. Describe what you see and extract key information.'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS analysis;
```

---

## If PDF File: Convert to Image First

### Step 1: Create the PDF-to-Image conversion stored procedure

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

### Step 2: Ask which pages to convert

```
Which pages of the PDF contain the charts/diagrams you want to analyze?
Options:
1. Convert all pages to images
2. Convert specific pages (e.g., 1, 3, 5)
3. Convert a page range (e.g., pages 1-10)
4. Convert just the first page to test
```

### Step 3: Convert PDF to images

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

### Step 4: Verify converted images

```sql
-- Refresh stage directory
ALTER STAGE db.schema.images_stage REFRESH;

-- List converted images
SELECT * FROM DIRECTORY(@db.schema.images_stage)
WHERE relative_path LIKE 'blueprint_%'
ORDER BY relative_path;
```

### Step 5: Analyze images with AI_COMPLETE

```sql
-- Analyze a single converted image
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  [
    {
      'role': 'user',
      'content': [
        {'type': 'image', 'image_url': {'url': TO_FILE('@db.schema.images_stage', 'blueprint_page_1.png')}},
        {'type': 'text', 'text': 'Analyze this blueprint/diagram. Identify all components, measurements, and annotations.'}
      ]
    }
  ],
  {'max_tokens': 4096}
) AS analysis;
```

---

## Step 2: Visual Analysis Prompts

**Ask** user what they want to analyze:

```
What would you like to analyze in the image?
Options:
1. Charts/graphs - extract data points, trends, labels
2. Blueprints/technical drawings - identify components, measurements
3. Diagrams/flowcharts - describe structure, relationships
4. General visual content - describe what you see
5. Custom analysis - I'll describe what I need
```

---

## Prompt Templates by Type

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
