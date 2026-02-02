# Google Drive Folder Download Automation

Automated workflow for downloading files from Google Drive folders and uploading to Snowflake stages.

## When to Use

- User is in CoCo Desktop environment
- Files are stored in Google Drive folders
- Need to bulk process multiple documents from a folder
- User wants automated download → upload workflow

## Prerequisites

- CoCo Desktop app with agentic browser capability
- User has access to Google Drive folder (authenticated)
- Snowflake stage available for upload

## Workflow

```
Start
  ↓
Step 1: Detect Environment & Browser State
  ↓
  ├─→ CoCo Desktop + Browser open with Google Drive → Offer automated download
  ├─→ CoCo Desktop + No browser/wrong page → Request Google Drive URL
  └─→ Non-Desktop environment → Fallback to manual upload
  ↓
Step 2: Open/Verify Google Drive Folder
  ↓
Step 3: Select All Files (Cmd+A)
  ↓
Step 4: Download Files (Right-click → Download)
  ↓
Step 5: Wait for Downloads to Complete
  ↓
Step 6: Upload to Snowflake Stage
  ↓
Step 7: Verify Upload & Return File Paths
```

## Step 1: Detect Environment & Browser State

### Check for CoCo Desktop Environment

```
First, check if running in CoCo Desktop:
- Check for browser tool availability
- Check for system environment indicators
```

**If NOT in CoCo Desktop:**
```
Google Drive bulk download automation requires CoCo Desktop with agentic browser.

Options:
1. Use openflow skill to set up Google Drive connector (recommended for continuous sync)
2. Manually download files and provide local file path
3. Share individual file links for processing
```

**If in CoCo Desktop:**

Check browser state using browser tools:
```
- browser_tabs() → Check for open tabs
- Look for tabs with Google Drive URLs (drive.google.com)
```

### Browser State Detection

**Case A: Browser open with Google Drive folder visible**
```
I can see you have a Google Drive folder open in your browser.

Would you like me to:
1. Download ALL files from the current folder
2. Select specific files to download
3. Navigate to a different Google Drive folder
```

**Case B: Browser open but not on Google Drive**
```
I can open Google Drive for you.

Please provide:
- Google Drive folder URL (e.g., https://drive.google.com/drive/folders/ABC123)

Or:
- Navigate to your folder manually and let me know when ready
```

**Case C: No browser open**
```
Let me open an agentic browser for Google Drive.

Please provide:
- Google Drive folder URL (e.g., https://drive.google.com/drive/folders/ABC123)
```

## Step 2: Open/Verify Google Drive Folder

### If URL Provided

```python
# Open Google Drive URL in agentic browser
browser_navigate(url=google_drive_url)

# Wait for page load
browser_wait_for(text="My Drive", timeout=10)

# Take snapshot to verify folder loaded
browser_snapshot()
```

**Check for authentication:**
- If login page appears → Ask user to authenticate manually
- If "Access Denied" → Verify user has permissions
- If folder loads → Proceed to next step

### If Browser Already on Folder

```python
# Verify we're on a Google Drive folder view
browser_snapshot()

# Check for Grid/List view with files
# Look for file selection checkboxes or file list
```

**Confirm with user:**
```
I can see [X] files in this folder:
- [List first 5-10 file names if visible]

Ready to download all [X] files?
```

## Step 3: Select All Files (Cmd+A)

### Ensure Files View is Active

```python
# Click on file area to ensure focus
# Avoid clicking on specific files to prevent opening
browser_click(element="file list area")

# Wait briefly
browser_wait_for(time=0.5)
```

### Select All Files

```python
# Send Cmd+A (Mac) or Ctrl+A (Windows)
browser_press_key(key="Meta+a")  # Mac
# or
browser_press_key(key="Control+a")  # Windows

# Wait for selection to register
browser_wait_for(time=1)

# Verify selection via snapshot
snapshot = browser_snapshot()
# Look for "X items selected" indicator
```

**Validation:**
- Check for "X items selected" text in UI
- Verify file checkboxes are checked
- If selection failed → Retry once, then ask user for manual selection

## Step 4: Download Files (Right-click → Download)

### Right-Click on Selected Files

```python
# Right-click on one of the selected files
# Google Drive will show context menu with "Download" option
browser_click(element="selected file", button="right")

# Wait for context menu
browser_wait_for(text="Download", timeout=3)

# Take snapshot to verify menu appeared
browser_snapshot()
```

### Click Download Option

```python
# Click "Download" in context menu
browser_click(element="Download menu item")

# Browser will initiate download
# For multiple files, Google Drive creates a ZIP file
```

**Expected Behavior:**
- Single file → Downloads directly
- Multiple files → Google Drive creates ZIP, then downloads
- Large folders → May show "Preparing download..." progress

### Handle Download Preparation

```python
# For large folders, wait for ZIP preparation
# Look for "Preparing your download..." or similar text
browser_wait_for(textGone="Preparing", timeout=60)

# Or check for download completion in browser
# Downloads appear in browser download bar
```

## Step 5: Wait for Downloads to Complete

### Monitor Download Status

**Option A: Check browser download status**
```python
# Some browsers expose download status via console/network
browser_console_messages()

# Or check for download completion indicator in UI
```

**Option B: Poll Downloads folder**
```bash
# Check for new files in Downloads folder
ls -lt ~/Downloads | head -10

# Look for ZIP file with recent timestamp
# Google Drive creates: "FolderName.zip" or similar
```

### Wait Strategy

```python
# Initial wait for download to start
time.sleep(5)

# Poll for file appearance
import time
download_file = None
for i in range(30):  # Max 30 seconds
    # Check for newest ZIP in Downloads
    files = glob.glob(os.path.expanduser("~/Downloads/*.zip"))
    if files:
        newest = max(files, key=os.path.getctime)
        # Check if file is still growing (incomplete download)
        size1 = os.path.getsize(newest)
        time.sleep(2)
        size2 = os.path.getsize(newest)
        if size1 == size2:
            # File stable, download complete
            download_file = newest
            break
    time.sleep(1)
```

### Extract ZIP if Needed

```bash
# If Google Drive created a ZIP
cd ~/Downloads
unzip -o "FolderName.zip" -d extracted_files/

# List extracted files
ls -la extracted_files/
```

## Step 6: Upload to Snowflake Stage

### Create Stage if Needed

```sql
CREATE STAGE IF NOT EXISTS <database>.<schema>.google_drive_docs
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');
```

**Ask user for stage details:**
```
Where should I upload these files in Snowflake?

Please provide:
- Database name (default: <current_database>)
- Schema name (default: <current_schema>)
- Stage name (default: google_drive_docs)
```

### Upload Files via PUT Command

**For extracted files:**
```bash
# Upload all files from extracted folder
snow sql -q "PUT file://~/Downloads/extracted_files/* @<db>.<schema>.<stage>/ AUTO_COMPRESS=FALSE;"

# Or upload individually
for file in ~/Downloads/extracted_files/*; do
    snow sql -q "PUT file://$file @<db>.<schema>.<stage>/ AUTO_COMPRESS=FALSE;"
done
```

**For single file (no ZIP):**
```bash
snow sql -q "PUT file://~/Downloads/<filename> @<db>.<schema>.<stage>/ AUTO_COMPRESS=FALSE;"
```

### Verify Upload

```sql
LIST @<database>.<schema>.<stage>;
```

```python
# Execute via snowflake_sql_execute
snowflake_sql_execute(
    sql=f"LIST @{database}.{schema}.{stage}",
    description="List uploaded files in stage"
)
```

## Step 7: Verify Upload & Return File Paths

### Display Results

```
Successfully downloaded and uploaded [X] files from Google Drive!

Files now available in Snowflake:
- @<database>.<schema>.<stage>/file1.pdf
- @<database>.<schema>.<stage>/file2.pdf
- @<database>.<schema>.<stage>/file3.pdf
...

Total: [X] files, [Y] MB

Ready to process these documents?
```

### Clean Up Downloads (Optional)

**Ask user:**
```
Would you like me to:
1. Keep downloaded files in ~/Downloads (for backup)
2. Delete downloaded files (already in Snowflake)
```

**If delete chosen:**
```bash
# Remove extracted files
rm -rf ~/Downloads/extracted_files/

# Remove ZIP file
rm ~/Downloads/FolderName.zip
```

### Next Steps

```
Files are now staged in Snowflake. What would you like to do next?

Options:
1. Extract structured data (AI_EXTRACT) from these documents
2. Parse full text content (AI_PARSE_DOCUMENT)
3. Analyze visual content (AI_COMPLETE)
4. Set up continuous processing pipeline

→ Return to main document-intelligence workflow (Step 1: Determine Extraction Goal)
```

## Error Handling

### Common Issues & Solutions

**Issue: Google Drive authentication expired**
```
I see the Google login page.

Please:
1. Log in to your Google account in the browser
2. Navigate to your folder
3. Let me know when ready to proceed
```

**Issue: Download blocked or failed**
```
Download appears to be blocked or failed.

Troubleshooting:
1. Check browser download permissions
2. Check Downloads folder permissions
3. Try downloading a single file manually to test
4. Check available disk space

Or fallback to:
- openflow skill for Google Drive connector
- Manual download and local file upload
```

**Issue: ZIP extraction failed**
```
Unable to extract downloaded ZIP file.

Please:
1. Manually extract ~/Downloads/FolderName.zip
2. Provide path to extracted folder
```

**Issue: Snowflake stage upload failed**
```
Error uploading to stage: <error_message>

Troubleshooting:
1. Verify stage exists and you have permissions
2. Check file formats are supported
3. Verify network connectivity to Snowflake
4. Check file sizes (max 50-100 MB per file)
```

### Fallback Options

If automation fails at any step:
```
Automated download encountered an issue.

Fallback options:
1. openflow skill → Set up Google Drive connector (recommended for reliability)
2. Manual download → Provide local file paths
3. Share links → Process individual files via shared links
4. Google Drive API → Use Python script with Drive API
```

## Platform-Specific Notes

### macOS
- Use `Cmd+A` (Meta+a) for select all
- Default Downloads: `~/Downloads/`
- File paths use forward slashes

### Windows
- Use `Ctrl+A` (Control+a) for select all
- Default Downloads: `%USERPROFILE%\Downloads\`
- File paths may use backslashes (convert to forward slashes for PUT)

### Linux
- Use `Ctrl+A` (Control+a) for select all
- Default Downloads: `~/Downloads/`
- File paths use forward slashes

## Security Considerations

- Never log Google credentials or tokens
- Clear browser session data if processing sensitive documents
- Use ENCRYPTION in Snowflake stages for sensitive data
- Consider using Snowflake's network policies for stage access
- Delete local downloads after upload if documents are confidential

## Performance Notes

- Google Drive ZIP creation time depends on:
  - Number of files (more files = longer prep)
  - Total size (larger folders = longer prep)
  - Google Drive server load
- Typical timing:
  - 10 files (~10 MB): 5-10 seconds
  - 50 files (~50 MB): 15-30 seconds
  - 100+ files or >100 MB: 1-5 minutes
- Snowflake PUT upload speed depends on:
  - File sizes
  - Network bandwidth
  - Snowflake region proximity

## Limitations

- Google Drive file limits:
  - Free accounts: Limited storage
  - Folder size: No hard limit, but large folders slow
  - File types: Google Docs/Sheets/Slides download as Office formats
- ZIP size limits: Browser may have download size limits
- Snowflake stage limits:
  - File size: Check stage configuration
  - PUT command: Works best with files <5 GB
