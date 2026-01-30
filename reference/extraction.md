# Extraction Sub-Skill

Structured field and table extraction using AI_EXTRACT with test-driven prompt refinement.

## When to Use

This sub-skill is triggered when the user selects:
- "Specific fields (names, dates, amounts, IDs)"
- "Tables with defined columns"

## AI_EXTRACT Constraints

**Inform the user before proceeding:**

| Constraint | Limit |
|------------|-------|
| Max file size | 100 MB |
| Max pages | 125 per call |
| Entity questions | 100 per call (512 tokens each) |
| Table questions | 10 per call (4096 tokens each) |
| Image dimensions | 50x50 to 10,000x10,000 pixels |

**Note:** 1 table question = 10 entity questions for quota purposes.

If documents exceed these limits, use chunking strategies (see "Large Document Strategies" section below).

## Cost Information

**IMPORTANT: Show this cost information to the user when first using AI_EXTRACT:**

```
AI_EXTRACT Pricing:
- Approximately 970 tokens per page
- Cost: 5 credits per million tokens
- Estimated cost per page: ~0.00485 credits

Example cost estimates:
- 1 page: ~0.005 credits
- 10 pages: ~0.05 credits
- 50 pages: ~0.24 credits
- 125 pages (max): ~0.61 credits

Formula: (pages × 970 tokens) / 1,000,000 × 5 credits

Tip: Use page optimization to extract only specific pages and reduce costs.
```

## Workflow

```
Define Fields → Select Test File → Run Test → Review Results
                                                    ↓
                                              Satisfied? ─Yes→ Next Steps
                                                    │
                                                   No
                                                    ↓
                                              Refine Prompts (max 3x)
                                                    ↓
                                              Still failing? → Fallback Method
```

---

## Step 1: Define Initial Extraction Fields

### First, Identify Document Type

**Ask** user what type of document they are processing:

```
What type of document are you extracting from?
Examples: invoice, receipt, contract, purchase order, resume, medical form, insurance claim, etc.
```

### Suggest Common Fields Based on Document Type

Based on the document type, **suggest commonly found fields** to help the user:

#### Invoices / Bills
```
Common fields found in invoices:
- invoice_number: Invoice or bill number
- invoice_date: Date the invoice was issued
- due_date: Payment due date
- vendor_name: Name of vendor/supplier
- vendor_address: Vendor's address
- customer_name: Bill-to customer name
- customer_address: Customer's billing address
- subtotal: Amount before tax
- tax_amount: Tax amount
- total_amount: Total amount due
- payment_terms: Payment terms (Net 30, etc.)
- po_number: Purchase order reference

Common tables:
- line_items: Description, Quantity, Unit Price, Amount
```

#### Receipts
```
Common fields found in receipts:
- store_name: Merchant/store name
- store_address: Store location
- receipt_date: Transaction date
- receipt_time: Transaction time
- receipt_number: Receipt/transaction number
- subtotal: Amount before tax
- tax_amount: Sales tax
- total_amount: Total paid
- payment_method: Cash, Credit Card, etc.
- card_last_four: Last 4 digits of card (if applicable)

Common tables:
- items_purchased: Item name, Quantity, Price
```

#### Contracts / Agreements
```
Common fields found in contracts:
- contract_title: Title or name of agreement
- effective_date: When contract takes effect
- expiration_date: When contract expires
- party_a_name: First party name
- party_a_address: First party address
- party_b_name: Second party name
- party_b_address: Second party address
- contract_value: Total contract value
- payment_terms: Payment schedule/terms
- governing_law: Jurisdiction/governing law
- signatory_names: Names of signers
- signature_date: Date signed
```

#### Purchase Orders (POs)
```
Common fields found in purchase orders:
- po_number: Purchase order number
- po_date: Date PO was created
- vendor_name: Supplier name
- vendor_address: Supplier address
- ship_to_address: Delivery address
- bill_to_address: Billing address
- requested_delivery_date: Expected delivery
- subtotal: Amount before tax
- shipping_cost: Shipping/freight charges
- tax_amount: Tax
- total_amount: Total PO value
- payment_terms: Payment terms
- buyer_name: Purchaser/buyer name

Common tables:
- order_items: Item number, Description, Quantity, Unit Price, Total
```

#### Resumes / CVs
```
Common fields found in resumes:
- candidate_name: Full name
- email: Email address
- phone: Phone number
- address: Location/address
- linkedin_url: LinkedIn profile
- summary: Professional summary/objective
- total_experience_years: Years of experience

Common tables:
- work_experience: Company, Title, Start Date, End Date, Description
- education: Institution, Degree, Field, Graduation Date
- skills: Skill name, Proficiency level
- certifications: Certification name, Issuer, Date
```

#### Medical Forms / Records
```
Common fields found in medical forms:
- patient_name: Patient full name
- date_of_birth: Patient DOB
- patient_id: Medical record number
- visit_date: Date of visit/service
- provider_name: Physician/provider name
- facility_name: Hospital/clinic name
- diagnosis_codes: ICD codes
- procedure_codes: CPT codes
- insurance_id: Insurance member ID
- insurance_name: Insurance company

Common tables:
- medications: Drug name, Dosage, Frequency
- lab_results: Test name, Value, Reference range
- procedures: Procedure, Date, Provider
```

#### Insurance Claims
```
Common fields found in insurance claims:
- claim_number: Claim reference number
- policy_number: Insurance policy number
- claimant_name: Name of claimant
- date_of_loss: Date incident occurred
- date_filed: Date claim was filed
- claim_type: Type of claim
- claim_amount: Amount claimed
- deductible: Deductible amount
- approved_amount: Amount approved
- claim_status: Status (pending, approved, denied)
- adjuster_name: Claims adjuster
```

#### Bank Statements
```
Common fields found in bank statements:
- account_holder: Account holder name
- account_number: Account number (may be masked)
- statement_period: Start and end date
- opening_balance: Balance at start
- closing_balance: Balance at end
- total_deposits: Sum of deposits
- total_withdrawals: Sum of withdrawals

Common tables:
- transactions: Date, Description, Amount, Balance
```

### Ask User for Fields

After suggesting common fields, **ask** user to confirm or customize:

```
What fields do you want to extract? Please provide field names and descriptions.

For each field, specify the extraction type:
1. Single Entity - one value (e.g., invoice_number, total_amount)
2. List - multiple values of same type (e.g., all signatory names)
3. Table - columnar data (e.g., line items with description, qty, price)

Example: 
- invoice_number (single): The invoice or bill number
- vendor_name (single): Name of the vendor or supplier
- total_amount (single): The total amount due
- line_items (table): Description, Quantity, Unit Price, Amount
```

### Build responseFormat Based on Extraction Types

**IMPORTANT:** AI_EXTRACT supports exactly 3 extraction types. Build the schema correctly for each type.

#### Type 1: Single Entity (one value)

For simple fields that extract a single value:

```sql
-- Simple format (question string)
responseFormat => {
  'invoice_number': 'What is the invoice number?',
  'vendor_name': 'What is the vendor name?',
  'total_amount': 'What is the total amount?'
}

-- OR Schema format (with type specification)
responseFormat => {
  'schema': {
    'type': 'object',
    'properties': {
      'invoice_number': {
        'type': 'string',
        'description': 'The invoice or bill number'
      },
      'vendor_name': {
        'type': 'string',
        'description': 'Name of the vendor or supplier'
      },
      'total_amount': {
        'type': 'string',
        'description': 'The total amount due'
      }
    }
  }
}
```

#### Type 2: List (array of values)

For extracting multiple values of the same type as an array:

```sql
responseFormat => {
  'schema': {
    'type': 'object',
    'properties': {
      'signatory_names': {
        'type': 'array',
        'items': {'type': 'string'},
        'description': 'Names of all people who signed the document'
      },
      'cc_recipients': {
        'type': 'array',
        'items': {'type': 'string'},
        'description': 'All CC email recipients'
      }
    }
  }
}
```

#### Type 3: Table (columnar data)

For extracting structured tabular data. **Must use `column_ordering`:**

```sql
responseFormat => {
  'schema': {
    'type': 'object',
    'properties': {
      'line_items': {
        'type': 'object',
        'description': 'Invoice line items table',
        'column_ordering': ['description', 'quantity', 'unit_price', 'amount'],
        'properties': {
          'description': {'type': 'array'},
          'quantity': {'type': 'array'},
          'unit_price': {'type': 'array'},
          'amount': {'type': 'array'}
        }
      }
    }
  }
}
```

### Combining All 3 Types in One Call

A single AI_EXTRACT call can combine all extraction types:

```sql
SELECT AI_EXTRACT(
  file => TO_FILE('@db.schema.stage', '<test_file>'),
  responseFormat => {
    'schema': {
      'type': 'object',
      'properties': {
        -- Type 1: Single entities
        'invoice_number': {
          'type': 'string',
          'description': 'Invoice number'
        },
        'vendor_name': {
          'type': 'string',
          'description': 'Vendor name'
        },
        'total_amount': {
          'type': 'string',
          'description': 'Total amount due'
        },
        -- Type 2: List
        'payment_methods': {
          'type': 'array',
          'items': {'type': 'string'},
          'description': 'Accepted payment methods'
        },
        -- Type 3: Table
        'line_items': {
          'type': 'object',
          'description': 'Line items table',
          'column_ordering': ['description', 'qty', 'price', 'amount'],
          'properties': {
            'description': {'type': 'array'},
            'qty': {'type': 'array'},
            'price': {'type': 'array'},
            'amount': {'type': 'array'}
          }
        }
      }
    }
  }
) AS result;
```

---

## Step 2: Select Test File

**Ask** user using `ask_user_question`:

```
How would you like to select the test file?
Options:
1. Use the first file in the stage
2. Let me pick a specific file from the stage
3. I'll provide the exact file path
```

**If "first file":**
```sql
-- Get first file from stage
SELECT relative_path 
FROM DIRECTORY(@db.schema.stage) 
WHERE relative_path LIKE '%.pdf' 
LIMIT 1;
```

**If "pick from stage":**
```sql
-- List available files
SELECT relative_path, size, last_modified
FROM DIRECTORY(@db.schema.stage)
WHERE relative_path LIKE '%.pdf'
ORDER BY last_modified DESC
LIMIT 10;
```
Then ask user to select from the list.

---

## Step 3: Run Test Extraction

Execute AI_EXTRACT on the single test file:

```sql
-- Test extraction on single file
SELECT AI_EXTRACT(
  file => TO_FILE('@db.schema.stage', '<selected_test_file>'),
  responseFormat => {
    'field1': 'question1',
    'field2': 'question2'
    -- ... user-defined fields
  }
) AS test_result;
```

**Display results to user in a readable format:**
```sql
SELECT 
  test_result:response:field1::STRING AS field1,
  test_result:response:field2::STRING AS field2
FROM (SELECT AI_EXTRACT(...) AS test_result);
```

---

## Step 4: Review Results

**Ask** user using `ask_user_question`:

```
Here are the test extraction results:

| Field | Extracted Value |
|-------|-----------------|
| invoice_number | INV-2024-001 |
| vendor_name | Acme Corp |
| total_amount | $1,234.56 |

Are you satisfied with these results?
Options:
1. Yes, proceed with full extraction
2. No, some fields need improvement
3. No, start over with different fields
```

**If "Yes":** Proceed to **Step 7: Next Steps**.

**If "Start over":** Return to Step 1.

**If "Some fields need improvement":** Continue to Step 5.

---

## Step 5: Prompt Refinement Loop (LLM-as-Judge)

**Ask** user which field(s) need improvement:

```
Which field(s) gave incorrect results?
(Select all that apply)
```

**For each problematic field, ask:**

```
For the field "[field_name]":
- Current extraction: "[actual_result]"
- What should the correct value be?
```

**Use Claude as LLM-judge to refine the prompt:**

```sql
-- LLM-as-judge prompt refinement
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  $You are an expert at crafting extraction prompts for document AI.

TASK: Improve an extraction question that didn't produce the expected result.

FIELD: [field_name]
ORIGINAL QUESTION: "[original_question]"
EXTRACTED VALUE: "[actual_result]"  
EXPECTED VALUE: "[expected_result]"
DOCUMENT TYPE: [document_type from Step 1]

Analyze why the original question may have failed and provide an improved question that would more accurately extract the expected value.

Consider:
- Is the question too vague or too specific?
- Should it reference a specific location in the document?
- Should it specify the format of the expected answer?
- Are there synonyms or alternative phrasings that might work better?

Return ONLY the improved question, nothing else.$
) AS improved_question;
```

**Update the responseFormat with refined question(s)** and retry test extraction.

### Iteration Limits

- Maximum **3 refinement iterations** per field
- After 3 attempts, **ask** user:
  ```
  We've tried 3 prompt variations for "[field_name]" without success.
  Options:
  1. Continue anyway with best result so far
  2. Skip this field for now
  3. Let me manually specify the extraction question
  4. Try fallback method (AI_PARSE_DOCUMENT + AI_COMPLETE)
  ```

**If "Try fallback method":** Proceed to Step 6.

---

## Step 6: Fallback Method (AI_PARSE_DOCUMENT + AI_COMPLETE)

When AI_EXTRACT doesn't produce accurate results even after prompt refinement, use this alternative approach:

1. **Parse the document text** using AI_PARSE_DOCUMENT
2. **Extract fields** from the parsed text using AI_COMPLETE with structured outputs

This method can be more effective for complex documents where AI_EXTRACT struggles.

### Step 6a: Parse document to get text content

```sql
-- Parse document using OCR or LAYOUT mode
SELECT AI_PARSE_DOCUMENT(
  TO_FILE('@db.schema.stage', '<test_file>'),
  {'mode': 'LAYOUT'}
):content::STRING AS document_text;
```

### Step 6b: Build JSON schema for structured output

Convert the user's fields into a JSON schema:

```sql
-- Example: For fields invoice_number, vendor_name, total_amount
-- Build this schema:
{
  'type': 'json',
  'schema': {
    'type': 'object',
    'properties': {
      'invoice_number': {
        'type': 'string',
        'description': 'The invoice or bill number'
      },
      'vendor_name': {
        'type': 'string', 
        'description': 'Name of the vendor or supplier'
      },
      'total_amount': {
        'type': 'string',
        'description': 'The total amount due'
      }
    },
    'required': ['invoice_number', 'vendor_name', 'total_amount']
  }
}
```

### Step 6c: Extract using AI_COMPLETE with structured outputs

```sql
-- Complete extraction with structured output
WITH parsed_doc AS (
  SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@db.schema.stage', '<test_file>'),
    {'mode': 'LAYOUT'}
  ):content::STRING AS document_text
)
SELECT AI_COMPLETE(
  'claude-3-5-sonnet',
  'Extract the following information from this document text. Return ONLY the requested fields in JSON format.

DOCUMENT TEXT:
' || document_text || '

Extract these fields:
- invoice_number: The invoice or bill number
- vendor_name: Name of the vendor or supplier  
- total_amount: The total amount due',
  {
    'response_format': {
      'type': 'json',
      'schema': {
        'type': 'object',
        'properties': {
          'invoice_number': {'type': 'string', 'description': 'The invoice or bill number'},
          'vendor_name': {'type': 'string', 'description': 'Name of the vendor or supplier'},
          'total_amount': {'type': 'string', 'description': 'The total amount due'}
        },
        'required': ['invoice_number', 'vendor_name', 'total_amount']
      }
    },
    'max_tokens': 4096
  }
) AS extraction_result
FROM parsed_doc;
```

### Step 6d: Compare results with AI_EXTRACT

Display both results to user:

```
Fallback extraction results:

| Field | AI_EXTRACT Result | AI_COMPLETE Result |
|-------|-------------------|-------------------|
| invoice_number | INV-2024-001 | INV-2024-001 |
| vendor_name | Acme | Acme Corporation |
| total_amount | 1234 | $1,234.56 |

Which result is more accurate?
Options:
1. AI_COMPLETE results are better - use fallback method
2. AI_EXTRACT results are better - stick with original
3. Neither is correct - try manual specification
```

### Step 6e: Batch processing with fallback method

```sql
-- Batch extraction using AI_PARSE_DOCUMENT + AI_COMPLETE
SELECT 
  relative_path AS file_path,
  PARSE_JSON(AI_COMPLETE(
    'claude-3-5-sonnet',
    'Extract the following information from this document text. Return ONLY the requested fields.

DOCUMENT TEXT:
' || AI_PARSE_DOCUMENT(
      TO_FILE('@db.schema.stage', relative_path),
      {'mode': 'LAYOUT'}
    ):content::STRING || '

Extract: invoice_number, vendor_name, total_amount',
    {
      'response_format': {
        'type': 'json',
        'schema': {
          'type': 'object',
          'properties': {
            'invoice_number': {'type': 'string'},
            'vendor_name': {'type': 'string'},
            'total_amount': {'type': 'string'}
          },
          'required': ['invoice_number', 'vendor_name', 'total_amount']
        }
      },
      'max_tokens': 4096
    }
  )) AS extracted_data
FROM DIRECTORY(@db.schema.stage)
WHERE relative_path LIKE '%.pdf';
```

### Structured Output Schema Examples

For different data types:

```sql
-- String fields
'field_name': {'type': 'string', 'description': 'Description of field'}

-- Number fields  
'amount': {'type': 'number', 'description': 'Numeric amount'}

-- Integer fields
'quantity': {'type': 'integer', 'description': 'Whole number quantity'}

-- Boolean fields
'is_paid': {'type': 'boolean', 'description': 'Whether invoice is paid'}

-- Enum fields (constrained values)
'status': {
  'type': 'string',
  'enum': ['pending', 'approved', 'rejected'],
  'description': 'Document status'
}

-- Array fields
'line_items': {
  'type': 'array',
  'items': {
    'type': 'object',
    'properties': {
      'description': {'type': 'string'},
      'quantity': {'type': 'integer'},
      'unit_price': {'type': 'number'},
      'total': {'type': 'number'}
    },
    'required': ['description', 'quantity', 'unit_price', 'total']
  },
  'description': 'List of line items'
}

-- Nested objects
'address': {
  'type': 'object',
  'properties': {
    'street': {'type': 'string'},
    'city': {'type': 'string'},
    'state': {'type': 'string'},
    'zip': {'type': 'string'}
  },
  'required': ['street', 'city', 'state', 'zip'],
  'description': 'Mailing address'
}
```

### When to prefer the fallback method

| Scenario | Recommendation |
|----------|----------------|
| Complex nested data (addresses, line items) | Use fallback - better at structured objects |
| Documents with unusual layouts | Use fallback - LAYOUT mode captures more context |
| Need specific data types (numbers, booleans) | Use fallback - schema enforces types |
| Simple flat fields | Stick with AI_EXTRACT - faster and simpler |
| High volume batch processing | Test both, compare accuracy and cost |

---

## Step 7: Next Steps After Validation

Once user is satisfied with test results, **ask** what they want to do next:

```
Test extraction validated successfully! What would you like to do next?
Options:
1. Extract from all files in the stage (batch processing)
2. Add more fields to extract
3. Store this single result to a table
4. Try on another test file first
5. Done - I only needed this one file
```

**Route based on response:**

| User Selection | Action |
|----------------|--------|
| Extract from all files | Proceed to batch extraction |
| Add more fields | Return to Step 1 with existing fields preserved |
| Store this result | Generate INSERT statement for single result |
| Try another test file | Return to Step 2 with same responseFormat |
| Done | End workflow, display final results |

**If "Add more fields":**
```
You currently have these fields defined:
- invoice_number
- vendor_name  
- total_amount

What additional fields would you like to extract?
```

---

## Large Document Strategies (>125 pages)

### Check for Large Files First

**Ask** user:
```
Can any of your files be longer than the page limit?
- AI_EXTRACT: 125 pages max
- AI_PARSE_DOCUMENT: 500 pages max
Options:
1. No - all files are within the limit
2. Yes - some files may exceed the limit
3. Not sure
```

### Page Optimization (to reduce cost and improve speed)

**Ask** user:
```
Would you like to process the entire document or only specific pages?
This can significantly reduce processing time and cost.

Options:
1. Process entire document (all pages)
2. Process only the first page (common for invoices/forms with header info)
3. Process specific page range (e.g., pages 1-5)
4. Process specific pages (e.g., pages 1, 3, 10)
```

### Page Extraction Stored Procedure

```sql
-- Create stage for extracted pages (with server-side encryption)
CREATE STAGE IF NOT EXISTS db.schema.extracted_pages_stage
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  COMMENT = 'Stage for extracted PDF pages';

-- Create page extraction stored procedure
CREATE OR REPLACE PROCEDURE db.schema.extract_pdf_pages(
    stage_name STRING, 
    file_name STRING, 
    dest_stage_name STRING,
    page_selection VARIANT  -- Can be: integer (single page), array of integers, or object with 'start' and 'end'
)
RETURNS VARIANT
LANGUAGE PYTHON
RUNTIME_VERSION = '3.9'
PACKAGES = ('snowflake-snowpark-python', 'pypdf2')
HANDLER = 'run'
AS
$
from PyPDF2 import PdfReader, PdfWriter
from snowflake.snowpark import Session
from snowflake.snowpark.file_operation import FileOperation
from io import BytesIO
import os
import json

def run(session: Session, stage_name: str, file_name: str, dest_stage_name: str, page_selection) -> dict:
    result = {
        "status": "success",
        "source_file": file_name,
        "total_pages": 0,
        "extracted_pages": [],
        "output_file": ""
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
            pdf_reader = PdfReader(BytesIO(pdf_data))
            num_pages = len(pdf_reader.pages)
            result["total_pages"] = num_pages
            
            # Determine which pages to extract (convert to 0-indexed)
            pages_to_extract = []
            
            if isinstance(page_selection, int):
                # Single page (1-indexed input)
                pages_to_extract = [page_selection - 1]
            elif isinstance(page_selection, list):
                # List of specific pages (1-indexed input)
                pages_to_extract = [p - 1 for p in page_selection]
            elif isinstance(page_selection, dict):
                # Range with 'start' and 'end' (1-indexed input)
                start = page_selection.get('start', 1) - 1
                end = min(page_selection.get('end', num_pages), num_pages)
                pages_to_extract = list(range(start, end))
            
            # Validate pages
            pages_to_extract = [p for p in pages_to_extract if 0 <= p < num_pages]
            
            if not pages_to_extract:
                return {"status": "error", "message": "No valid pages to extract"}
            
            # Create new PDF with selected pages
            writer = PdfWriter()
            for page_num in pages_to_extract:
                writer.add_page(pdf_reader.pages[page_num])
                result["extracted_pages"].append(page_num + 1)  # Return 1-indexed
            
            # Generate output filename
            base_name = os.path.splitext(file_name)[0]
            if len(pages_to_extract) == 1:
                output_filename = f'{base_name}_page_{pages_to_extract[0] + 1}.pdf'
            else:
                output_filename = f'{base_name}_pages_{pages_to_extract[0] + 1}_to_{pages_to_extract[-1] + 1}.pdf'
            
            output_path = os.path.join('/tmp/', output_filename)
            
            with open(output_path, 'wb') as output_file:
                writer.write(output_file)
            
            # Upload to destination stage
            FileOperation(session).put(
                f"file://{output_path}",
                dest_stage_name,
                auto_compress=False
            )
            
            result["output_file"] = output_filename
            
            # Cleanup
            os.remove(file_path)
            os.remove(output_path)
                
        return result
        
    except Exception as e:
        return {"status": "error", "message": str(e)}
$;
```

### Extract pages based on user selection

```sql
-- Option 2: First page only
CALL db.schema.extract_pdf_pages(
  '@db.schema.my_stage',
  'invoice.pdf',
  '@db.schema.extracted_pages_stage',
  1  -- Single page (1-indexed)
);

-- Option 3: Page range (e.g., pages 1-5)
CALL db.schema.extract_pdf_pages(
  '@db.schema.my_stage',
  'invoice.pdf',
  '@db.schema.extracted_pages_stage',
  {'start': 1, 'end': 5}  -- Range (1-indexed, inclusive)
);

-- Option 4: Specific pages (e.g., pages 1, 3, 10)
CALL db.schema.extract_pdf_pages(
  '@db.schema.my_stage',
  'invoice.pdf',
  '@db.schema.extracted_pages_stage',
  [1, 3, 10]  -- Array of pages (1-indexed)
);
```

### Then run AI_EXTRACT on extracted pages

```sql
-- Extract from the optimized PDF (fewer pages = faster + cheaper)
SELECT AI_EXTRACT(
  file => TO_FILE('@db.schema.extracted_pages_stage', '<output_file_from_procedure>'),
  responseFormat => {
    'invoice_number': 'What is the invoice number?',
    'vendor_name': 'What is the vendor name?',
    'total_amount': 'What is the total amount?'
  }
) AS result;
```

---

## Handoff to Batch Extraction

**Example handoff context:**
```
Proceeding to full extraction with validated schema:

Stage: @db.schema.stage
File pattern: %.pdf
Validated responseFormat:
{
  'invoice_number': 'What is the invoice or bill number shown at the top of the document?',
  'vendor_name': 'What is the name of the company or vendor issuing this invoice?',
  'total_amount': 'What is the final total amount due, including taxes?'
}

Test results confirmed accurate on: sample_invoice.pdf
```

---

## Next Steps: Post-Processing

After extraction is complete, **ask the user** what they want to do next:

→ **Load `reference/pipeline.md`** for post-processing options:
- One-time extraction (done)
- Store results in a Snowflake table
- Set up a continuous processing pipeline with streams and tasks

The pipeline sub-skill contains templates for extraction pipelines including chunking and full-document patterns.
