# Task 4: Python Package Stats Aggregator (DA-1266)

## Overview

This is the **main workflow** that orchestrates monthly collection of Python package download statistics. It reads a list of packages from n8n Data Tables, calls the PyPI Stats Sub-Workflow (Task 3) for each enabled package, and stores results in Google Sheets.

**Purpose**: Automate monthly tracking of Couchbase Python package downloads from PyPI

**Schedule**: Runs automatically on the 1st of each month at 10:00 AM UTC

**Prerequisites**:
- Task 3 (PyPI Sub-Workflow) must be created and active first
- Google Sheets credential configured in n8n
- Data table `python_packages` created in n8n

## Before You Start - Important!

### Understanding the Files

This folder contains **two different types of files**:

| File | Type | Purpose | Can Import to n8n? |
|------|------|---------|-------------------|
| `package-stats-aggregator.json` | **Workflow** | The actual n8n workflow | ✅ **YES** - Import this! |
| `data-table-schema.json` | **Documentation** | Instructions for creating data table | ❌ **NO** - Read & follow |

### Common Mistake ⚠️

**DO NOT try to import `data-table-schema.json` into n8n canvas!**

- This file is **documentation only**
- It tells you **how to manually create** the data table
- If you paste it into n8n canvas, nothing will happen (this is expected)

### The Correct Process

1. **First**: Read `data-table-schema.json` to understand what you need
2. **Second**: Manually create the data table in n8n (step-by-step below)
3. **Third**: Import `package-stats-aggregator.json` as the workflow
4. **Fourth**: Configure credentials and links

## Workflow Architecture

This workflow demonstrates the **orchestrator pattern**:
- Main workflow controls the flow
- Reads configuration from data table
- Calls sub-workflow multiple times (once per package)
- Aggregates and stores results

```
┌─────────────────────────────────────────────┐
│  Package Stats Aggregator (This Workflow)  │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │ 1. Trigger (Manual/Schedule)          │ │
│  │    • Manual for testing               │ │
│  │    • Schedule: 1st of month, 10 AM   │ │
│  └───────────────────────────────────────┘ │
│              ↓                              │
│  ┌───────────────────────────────────────┐ │
│  │ 2. Calculate Previous Month Date      │ │
│  │    • Run date: Feb 1, 2025            │ │
│  │    • Data for: Jan 2025 (2025-01)    │ │
│  └───────────────────────────────────────┘ │
│              ↓                              │
│  ┌───────────────────────────────────────┐ │
│  │ 3. Read Data Table                    │ │
│  │    Table: python_packages             │ │
│  │    Filter: is_enabled = true          │ │
│  │    Results:                           │ │
│  │      - couchbase                      │ │
│  │      - langchain-couchbase            │ │
│  └───────────────────────────────────────┘ │
│              ↓                              │
│  ┌───────────────────────────────────────┐ │
│  │ 4. Loop: Process Each Package         │ │
│  │    (Split in Batches - 1 at a time)  │ │
│  │              ↓                        │ │
│  │    ┌──────────────────────┐          │ │
│  │    │ Merge date + package │          │ │
│  │    └──────────────────────┘          │ │
│  │              ↓                        │ │
│  │    ┌──────────────────────┐          │ │
│  │    │ Call Sub-Workflow    │──────────┼─┼─> Task 3
│  │    │ (Task 3)             │          │ │   PyPI Stats
│  │    │ Input:               │          │ │
│  │    │  - package_name      │          │ │
│  │    │  - date (2025-01)    │          │ │
│  │    └──────────────────────┘          │ │
│  │              ↓                        │ │
│  │    ┌──────────────────────┐          │ │
│  │    │ Receive results:     │<─────────┼─┘
│  │    │  - downloads         │          │
│  │    │  - unique_downloads  │          │
│  │    └──────────────────────┘          │
│  │              ↓                        │ │
│  │    ┌──────────────────────┐          │ │
│  │    │ Write to Google      │          │ │
│  │    │ Sheets (dynamic tab) │          │ │
│  │    └──────────────────────┘          │
│  │              ↓                        │ │
│  │    [More packages?] ──→ Loop back    │ │
│  └───────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Step-by-Step Setup Guide

### Phase 1: Create the Data Table (MANUAL SETUP)

This is the most important step - the workflow **cannot work** without this table.

#### Step 1.1: Find Data Tables in n8n

1. Log into n8n: <https://kaustavcouchbase.app.n8n.cloud>
2. Look for **one of these** in the left sidebar or top menu:
   - "Data" or "Data Tables"
   - "Tables" (might be under Settings)
   - "Database" or "Storage"
   - If you can't find it, try clicking your profile icon → Settings → Data

**What you're looking for**: A menu option that lets you create/manage tables for storing data within n8n.

#### Step 1.2: Create the Table

1. Click **"Create Table"** or **"New Table"** or **"Add Table"**
2. You should see a form asking for:
   - Table name
   - Columns to add

#### Step 1.3: Name the Table

**Important**: Use this exact name (case-sensitive):

```
python_packages
```

**Why this name matters**: The workflow JSON is hard-coded to look for a table named `python_packages`. If you use a different name, the workflow will fail.

#### Step 1.4: Add Columns

Click "Add Column" three times and configure each:

**Column 1:**
- Name: `package_name`
- Type: `Text` (or `String`)
- Description (optional): "Python package name on PyPI"

**Column 2:**
- Name: `tab_name`
- Type: `Text` (or `String`)
- Description (optional): "Google Sheets tab for this package"

**Column 3:**
- Name: `is_enabled`
- Type: `Boolean` (or `Checkbox` or `True/False`)
- Description (optional): "Whether to track this package"
- Default: `true` (checked)

#### Step 1.5: Save the Table

Click **"Create"** or **"Save"** to create the empty table.

**✓ Validation**: You should now see `python_packages` in your list of data tables.

#### Step 1.6: Add Initial Data

Now add two rows to the table:

**Row 1:**
1. Click **"Add Row"** or **"+"**
2. Fill in:
   - `package_name`: `couchbase`
   - `tab_name`: `python-couchbase`
   - `is_enabled`: ✓ (checked/true)
3. Click **"Save"** or **"Add"**

**Row 2:**
1. Click **"Add Row"** or **"+"** again
2. Fill in:
   - `package_name`: `langchain-couchbase`
   - `tab_name`: `python-langchain`
   - `is_enabled`: ✓ (checked/true)
3. Click **"Save"** or **"Add"**

**✓ Validation**: Your data table should now show 2 rows with both packages enabled.

#### Step 1.7: Test the Data Table

To verify the table works:

1. Try adding a test row with any package name
2. Toggle the `is_enabled` checkbox on/off
3. Delete the test row
4. Confirm you still have your 2 main packages

**✓ Checkpoint**: Data table created and populated! ✅

---

### Phase 2: Prepare Google Sheets

#### Step 2.1: Open the Spreadsheet

Open: <https://docs.google.com/spreadsheets/d/1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8/>

#### Step 2.2: Create Required Tabs

Create two new tabs (if they don't exist):

**Tab 1: `python-couchbase`**
1. Click **"+"** at the bottom to add new sheet
2. Right-click sheet tab → Rename → `python-couchbase`
3. Add column headers in row 1:
   - A1: `Date`
   - B1: `Package Name`
   - C1: `Downloads`
   - D1: `Unique Downloads`

**Tab 2: `python-langchain`**
1. Click **"+"** to add another sheet
2. Right-click sheet tab → Rename → `python-langchain`
3. Add same column headers as above

**✓ Validation**: You should have 2 tabs with matching names from your data table's `tab_name` column.

#### Step 2.3: Share Permissions

If using OAuth2:
- The Google account you authorize in n8n must have edit access to this spreadsheet

If using Service Account:
- Share the spreadsheet with the service account email (ends with `@...iam.gserviceaccount.com`)
- Grant "Editor" permission

**✓ Checkpoint**: Google Sheets ready! ✅

---

### Phase 3: Import the Workflow

Now we import the actual workflow JSON file.

#### Step 3.1: Locate the Workflow File

In this folder, find the file:
```
package-stats-aggregator.json
```

**This is the one you import** (NOT the schema file!)

#### Step 3.2: Import into n8n

1. In n8n, click **"Workflows"** in the left sidebar
2. Click **"Import from File"** (or **"Add workflow"** → **"Import"**)
3. **Upload** `package-stats-aggregator.json` (or copy/paste the JSON content)
4. Click **"Import"**

#### Step 3.3: Verify Import

After import, you should see:
- Workflow name: "Python Package Stats Aggregator"
- Multiple nodes visible on the canvas
- Nodes: Manual Trigger, Schedule Trigger, Code nodes, Data Table node, Execute Workflow node, Google Sheets node, etc.

**✓ Validation**: Count the nodes - you should see approximately 9-10 nodes connected together.

**✓ Checkpoint**: Workflow imported! ✅

---

### Phase 4: Configure the Workflow

Now we need to configure each node properly.

#### Step 4.1: Open the Workflow for Editing

1. Double-click the workflow name or click "Edit"
2. You should see the workflow canvas with all nodes

#### Step 4.2: No Changes Needed for Triggers

The **Manual Trigger** and **Schedule Trigger** nodes are pre-configured:
- Schedule: `0 10 1 * *` (1st of month, 10:00 AM UTC)
- No credentials needed

**✓ Skip these nodes** - they're ready to go!

#### Step 4.3: Verify "Calculate Previous Month Date" Node

1. Click the **"Calculate Previous Month Date"** node
2. This is a Code node - check that it contains date calculation logic
3. **No changes needed** - this node is pre-configured

#### Step 4.4: Configure "Read from Data Table" Node

1. Click the **"Read from Data Table"** node
2. Settings to verify:
   - **Operation**: `Get All` or `Read All` or `List`
   - **Table**: Select `python_packages` (dropdown)
   - **Filter** (if available): `is_enabled = true`
   - **Return All**: `true` (or similar option)

**⚠️ Important**: If you don't see `python_packages` in the dropdown, go back to Phase 1 and create the data table!

**✓ Validation**: Execute this node alone (click "Execute Node") and verify it returns 2 items (your packages).

#### Step 4.5: No Changes for Split in Batches

The **"Split in Batches"** node is pre-configured:
- Batch Size: 1 (process one package at a time)

**✓ Skip this node** - ready to go!

#### Step 4.6: No Changes for Merge Date Node

The **"Merge Date with Package Data"** node is a Code node - pre-configured.

**✓ Skip this node** - ready to go!

#### Step 4.7: Configure "Call PyPI Stats Sub-Workflow" Node

This is the most important configuration!

1. Click the **"Call PyPI Stats Sub-Workflow"** node (or "Execute Workflow" node)
2. Settings to configure:
   - **Source**: `Database` (workflow from n8n)
   - **Workflow**:
     - **IMPORTANT**: You need to **select Task 3 workflow**
     - Look for "PyPI Stats Sub-Workflow" in the dropdown
     - If you don't see it, Task 3 is not created yet!
   - **Fields to Send**:
     - `package_name`: `={{ $json.package_name }}`
     - `date`: `={{ $json.date }}`

**⚠️ Critical**: If Task 3 workflow doesn't appear in the dropdown:
1. Go create Task 3 first (see `workflows/3-pypistats-subworkflow/`)
2. Make sure Task 3 is **saved** and **active**
3. Come back and select it here

**✓ Validation**: The dropdown should show "PyPI Stats Sub-Workflow" or similar.

#### Step 4.8: No Changes for Merge Results Node

The **"Merge Results"** node is a Code node - pre-configured.

**✓ Skip this node** - ready to go!

#### Step 4.9: Configure "Google Sheets - Append Stats" Node

1. Click the **"Google Sheets - Append Stats"** node
2. **Credential**: Click the dropdown
   - If you already have a Google Sheets OAuth2 credential, select it
   - If not, click **"Create New Credential"**:
     - Choose "Google Sheets OAuth2 API"
     - Follow the OAuth flow to authorize
     - Grant permissions
3. Settings to verify:
   - **Operation**: `Append`
   - **Document**:
     - **ID Mode** (if available): Select "ID"
     - **Document ID**: `1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8`
   - **Sheet**:
     - **Mode**: `Expression` or `Map`
     - **Value**: `={{ $json.tab_name }}` (dynamic - uses tab from data table!)
   - **Columns**:
     - `Date`: `={{ $json.date }}`
     - `Package Name`: `={{ $json.package_name }}`
     - `Downloads`: `={{ $json.downloads }}`
     - `Unique Downloads`: `={{ $json.unique_downloads }}`

**⚠️ Important**: The sheet name uses `{{ $json.tab_name }}` which is **dynamic** - it will write to different tabs based on the data table!

**✓ Validation**: The node should show your Google account credential selected.

#### Step 4.10: No Changes for "More Items to Process?" Node

This is an IF node that loops back - pre-configured.

**✓ Skip this node** - ready to go!

**✓ Checkpoint**: All nodes configured! ✅

---

### Phase 5: Test the Workflow

Before scheduling, we must test manually!

#### Step 5.1: Verify Prerequisites Checklist

Before testing, confirm:

- [ ] Data table `python_packages` exists with 2 rows
- [ ] Both packages have `is_enabled` = true
- [ ] Google Sheets has tabs `python-couchbase` and `python-langchain`
- [ ] Task 3 (PyPI Sub-Workflow) is created and active
- [ ] Google Sheets credential is configured
- [ ] Execute Workflow node is linked to Task 3

#### Step 5.2: Test with Manual Trigger

1. Click the **"Manual Trigger"** node
2. Click **"Execute Workflow"** (button at top or bottom)
3. Watch the execution flow through nodes

**Expected behavior:**
- Nodes light up green as they execute
- You'll see data flowing through
- Total execution time: ~30-60 seconds (depends on PyPI response time)

#### Step 5.3: Check for Errors

**If nodes turn red:**
- Click the red node to see error message
- Common errors:
  - "Table not found" → Go back to Phase 1
  - "Workflow not found" → Task 3 not created/active
  - "Unauthorized" → Google Sheets credential issue
  - "No data" → Data table is empty

**If execution hangs:**
- Wait up to 2 minutes (PyPI might be slow)
- Check Task 3 sub-workflow logs separately

#### Step 5.4: Verify Results in Google Sheets

1. Open the Google Sheets spreadsheet
2. Check tab `python-couchbase`:
   - Should have a new row with today's date and download numbers
3. Check tab `python-langchain`:
   - Should also have a new row

**Expected output example:**

**python-couchbase tab:**
| Date | Package Name | Downloads | Unique Downloads |
|------|--------------|-----------|------------------|
| 2025-01 | couchbase | 45123 | 38456 |

**python-langchain tab:**
| Date | Package Name | Downloads | Unique Downloads |
|------|--------------|-----------|------------------|
| 2025-01 | langchain-couchbase | 8234 | 6890 |

**✓ Validation**: You see new data rows in both tabs with reasonable numbers (not null).

#### Step 5.5: Review Execution Logs

1. In n8n, go to **"Executions"** tab
2. Click the most recent execution
3. Review each node's output:
   - Data Table node: Should show 2 items
   - Sub-workflow calls: Should show download numbers
   - Google Sheets: Should show success

**✓ Checkpoint**: Manual test passed! ✅

---

### Phase 6: Activate the Schedule

Now that testing works, enable automatic scheduling.

#### Step 6.1: Activate the Workflow

1. At the top of the workflow editor, find the **"Active"** toggle
2. Switch it **ON** (should turn green/blue)
3. You should see "Workflow is active" or similar confirmation

#### Step 6.2: Verify Schedule

1. The **Schedule Trigger** node should now be active
2. It will run automatically on: **1st of every month at 10:00 AM UTC**
3. Cron expression: `0 10 1 * *`

#### Step 6.3: Calculate Next Run

To know when it will run next:
- If today is before the 1st: Next run is the 1st of this month
- If today is on/after the 1st: Next run is the 1st of next month
- Time: 10:00 AM UTC (convert to your timezone)

**✓ Checkpoint**: Workflow is scheduled! ✅

---

## Data Table Details

### Why Use Data Tables?

**Benefits:**
- **No code changes**: Add/remove packages without editing workflow
- **Easy management**: Enable/disable packages via checkbox
- **Version controlled**: Changes tracked within n8n
- **No external database**: Built into n8n Cloud

### Table Schema

| Column Name | Type | Required | Description |
|-------------|------|----------|-------------|
| `package_name` | Text | Yes | Python package name exactly as it appears on PyPI (e.g., "couchbase", "langchain-couchbase") |
| `tab_name` | Text | Yes | Google Sheets tab where results will be written (e.g., "python-couchbase") |
| `is_enabled` | Boolean | Yes | Checkbox - checked means track this package, unchecked means skip |

### Adding New Packages

When Couchbase releases new Python packages:

1. Open n8n → Data → Tables → `python_packages`
2. Click **"Add Row"**
3. Fill in:
   - `package_name`: New package (must exist on <https://pypi.org>)
   - `tab_name`: New Google Sheets tab name
   - `is_enabled`: ✓ (checked)
4. Save
5. In Google Sheets, create the new tab with matching name
6. **Test**: Run workflow manually to verify new package works
7. Next scheduled run will automatically include it

**Example - Adding a third package:**
- `package_name`: `acouchbase`
- `tab_name`: `python-acouchbase`
- `is_enabled`: ✓

### Disabling Packages Temporarily

To stop tracking a package without deleting:

1. Open the data table
2. Find the package row
3. **Uncheck** the `is_enabled` checkbox
4. Save
5. Next run will skip this package (historical data remains in Google Sheets)

### Removing Packages

To permanently stop tracking:

1. Open the data table
2. Find the package row
3. Click **"Delete"** or trash icon
4. Confirm deletion
5. Package will not be tracked in future runs
6. Historical data in Google Sheets is preserved

## Google Sheets Details

### Spreadsheet Information

- **URL**: <https://docs.google.com/spreadsheets/d/1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8/>
- **Owner**: Couchbase DevEx team
- **Purpose**: Historical tracking of Python package downloads

### Tab Structure

Each package has its own tab with this structure:

| Column | Description | Example |
|--------|-------------|---------|
| Date | Month in yyyy-MM format | 2025-01 |
| Package Name | PyPI package name | couchbase |
| Downloads | Total downloads with mirrors | 45123 |
| Unique Downloads | Downloads without mirrors | 38456 |

### Tab Naming Convention

- Tab names come from the data table's `tab_name` column
- Must match **exactly** (case-sensitive!)
- Use kebab-case: `python-couchbase`, `python-langchain`

### Permissions Setup

**For OAuth2 (Recommended):**
1. When configuring Google Sheets credential in n8n, authorize with your Google account
2. Make sure that account has edit access to the spreadsheet
3. No additional sharing needed

**For Service Account:**
1. Get service account email (ends with `@...iam.gserviceaccount.com`)
2. Open Google Sheets → Click "Share"
3. Add service account email
4. Grant "Editor" permission
5. Save

## Workflow Node Details

### Node-by-Node Breakdown

#### 1. Manual Trigger
- **Type**: Trigger
- **Purpose**: Allows manual testing
- **When**: Click "Execute Workflow" button
- **Configuration**: None needed

#### 2. Schedule Trigger
- **Type**: Trigger
- **Purpose**: Automatic monthly execution
- **When**: 1st of month, 10:00 AM UTC
- **Cron**: `0 10 1 * *`
- **Configuration**: Pre-configured

#### 3. Calculate Previous Month Date
- **Type**: Code
- **Purpose**: Calculate previous month in yyyy-MM format
- **Why**: Running on the 1st means collecting data for the month that just ended
- **Logic**:
  ```javascript
  const now = new Date();
  const lastMonth = new Date(now.getFullYear(), now.getMonth() - 1, 1);
  const dateStr = lastMonth.toISOString().slice(0, 7); // "2025-01"
  ```
- **Output**: `{ date: "2025-01" }`

#### 4. Read from Data Table
- **Type**: n8n Data Table
- **Purpose**: Load list of packages to track
- **Table**: `python_packages`
- **Filter**: `is_enabled = true`
- **Output**: Array of package objects
  ```json
  [
    {
      "package_name": "couchbase",
      "tab_name": "python-couchbase",
      "is_enabled": true
    },
    {
      "package_name": "langchain-couchbase",
      "tab_name": "python-langchain",
      "is_enabled": true
    }
  ]
  ```

#### 5. Split in Batches
- **Type**: Loop
- **Purpose**: Process packages one at a time
- **Batch Size**: 1
- **Why**: Ensures sub-workflow is called sequentially, not all at once

#### 6. Merge Date with Package Data
- **Type**: Code
- **Purpose**: Combine date from step 3 with package from step 4
- **Output**:
  ```json
  {
    "package_name": "couchbase",
    "tab_name": "python-couchbase",
    "is_enabled": true,
    "date": "2025-01"
  }
  ```

#### 7. Call PyPI Stats Sub-Workflow
- **Type**: Execute Workflow
- **Purpose**: Call Task 3 to fetch PyPI stats
- **Workflow**: "PyPI Stats Sub-Workflow"
- **Input Parameters**:
  - `package_name`: `{{ $json.package_name }}`
  - `date`: `{{ $json.date }}`
- **Expected Output from Sub-Workflow**:
  ```json
  {
    "package_name": "couchbase",
    "date": "2025-01",
    "downloads": 45123,
    "unique_downloads": 38456,
    "success": true
  }
  ```

#### 8. Merge Results
- **Type**: Code
- **Purpose**: Combine sub-workflow results with package metadata
- **Output**:
  ```json
  {
    "package_name": "couchbase",
    "tab_name": "python-couchbase",
    "date": "2025-01",
    "downloads": 45123,
    "unique_downloads": 38456,
    "dateCollected": "2025-02-01"
  }
  ```

#### 9. Google Sheets - Append Stats
- **Type**: Google Sheets
- **Operation**: Append
- **Spreadsheet**: `1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8`
- **Sheet**: `={{ $json.tab_name }}` (dynamic!)
- **Columns Mapped**:
  - Date → `{{ $json.date }}`
  - Package Name → `{{ $json.package_name }}`
  - Downloads → `{{ $json.downloads }}`
  - Unique Downloads → `{{ $json.unique_downloads }}`

#### 10. More Items to Process?
- **Type**: IF condition
- **Purpose**: Check if more packages in batch
- **Condition**: `{{ $('Split in Batches').context.noItemsLeft }}` == false
- **True**: Loop back to Split in Batches
- **False**: Workflow complete

### Data Flow Example

**Full execution for couchbase package:**

```
1. Trigger → Executes
2. Calculate Date → Returns: { date: "2025-01" }
3. Read Data Table → Returns: [{ package_name: "couchbase", ... }, ...]
4. Split in Batches → Processes first item: { package_name: "couchbase", ... }
5. Merge Date → Combines: { package_name: "couchbase", date: "2025-01", ... }
6. Call Sub-Workflow → Fetches PyPI stats → Returns: { downloads: 45123, ... }
7. Merge Results → Combines all data
8. Write to Sheets → Appends row to "python-couchbase" tab
9. More Items? → Yes, loop back to step 4 for "langchain-couchbase"
```

## Scheduling Logic

### Why Run on 1st of Month?

**Reason**: Collect complete statistics for the previous month.

**Example:**
- **Run Date**: February 1, 2025
- **Data Requested**: January 2025 (`2025-01`)
- **Includes**: All downloads from Jan 1 - Jan 31

### Cron Expression Breakdown

`0 10 1 * *`

| Position | Value | Meaning |
|----------|-------|---------|
| 1 | `0` | Minute: 0 (on the hour) |
| 2 | `10` | Hour: 10 (10 AM) |
| 3 | `1` | Day: 1 (first day of month) |
| 4 | `*` | Month: Every month |
| 5 | `*` | Day of week: Any day |

**Translation**: Run at 10:00 AM UTC on the 1st day of every month.

### Timezone Considerations

- **UTC** is used for consistency across team members
- Convert to your timezone:
  - **PST**: 2:00 AM on the 1st
  - **EST**: 5:00 AM on the 1st
  - **IST**: 3:30 PM on the 1st

## Common Mistakes & FAQ

### Mistake 1: Trying to Import Schema File

**Problem**: Pasting `data-table-schema.json` into n8n canvas does nothing.

**Why**: It's documentation, not a workflow.

**Solution**: Read the schema file, then manually create the data table. Import `package-stats-aggregator.json` instead.

### Mistake 2: Data Table Not Found

**Problem**: Workflow fails with "Table 'python_packages' not found"

**Why**: Data table wasn't created or has wrong name.

**Solution**:
1. Go to n8n → Data → Tables
2. Verify `python_packages` exists (exact spelling, lowercase, underscore)
3. If missing, follow Phase 1 above

### Mistake 3: Sub-Workflow Not Found

**Problem**: "Execute Workflow" node shows "Workflow not found"

**Why**: Task 3 not created or not active.

**Solution**:
1. Create Task 3 first (see `workflows/3-pypistats-subworkflow/`)
2. Make sure Task 3 is **saved**
3. Make sure Task 3 is **active** (toggle on)
4. Come back and select it in the dropdown

### Mistake 4: Wrong Google Sheets Tab

**Problem**: Data written to wrong tab or "Sheet not found" error

**Why**: Tab name in data table doesn't match actual tab in Google Sheets.

**Solution**:
1. Check data table `tab_name` values (e.g., "python-couchbase")
2. Check Google Sheets tab names (exact match required!)
3. Rename tab in Sheets or update data table to match

### Mistake 5: No Data in Google Sheets

**Problem**: Workflow runs successfully but no data appears.

**Why**: Multiple possible causes.

**Solution**:
1. Check sub-workflow (Task 3) logs - did it return data?
2. Verify Google Sheets credential has edit permissions
3. Check spreadsheet ID is correct: `1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8`
4. Ensure tabs exist before running workflow

### FAQ: How Do I Know If My Data Table Is Working?

**Test it:**
1. Create a simple workflow with just a Manual Trigger and Read Data Table node
2. Execute the workflow
3. Check output - you should see your 2 packages
4. If you see them, the table works!

### FAQ: Where Exactly Is "Data" or "Tables" in n8n?

**n8n Cloud Navigation (may vary by version):**

Try these locations:
1. **Left sidebar** → Look for "Data" or "Tables" icon
2. **Settings** → "Data" or "Storage" section
3. **Profile menu** (top right) → "Data Tables"
4. **Workflows menu** → "Data" submenu

If still can't find it:
- Check n8n documentation: <https://docs.n8n.io/data/>
- Your n8n version may not have Data Tables yet (check version)

### FAQ: Can I Test Without Creating All Google Sheets Tabs?

**Yes, but:**
- Comment out or disable the Google Sheets node temporarily
- OR create just one tab and disable all other packages in data table
- Test with only one enabled package first

### FAQ: What If PyPI Stats (Task 3) Returns Null?

**Possible reasons:**
- Package doesn't exist on PyPI (typo in name?)
- Date is too old (PyPI Stats limits to ~3 months)
- pypistats.org is down or changed structure
- Task 3 parsing logic needs updating

**Debug:**
1. Test Task 3 independently with known package (e.g., "requests")
2. Check Task 3 logs for errors
3. Visit <https://pypistats.org/packages/couchbase> manually to verify data exists

## Troubleshooting

### Issue: Workflow Execution Hangs

**Symptoms**: Workflow starts but doesn't complete, no errors shown.

**Possible Causes:**
- Sub-workflow (Task 3) is taking too long or hanging
- pypistats.org is slow or unresponsive
- Network timeout

**Solution:**
1. Check Task 3 execution logs separately
2. Increase timeout in Execute Workflow node (if option available)
3. Test Task 3 independently to verify it works
4. Try again later if pypistats.org is having issues

### Issue: "Unauthorized" or "Forbidden" from Google Sheets

**Symptoms**: Error message about permissions when writing to sheets.

**Possible Causes:**
- Google Sheets credential not configured
- Credential expired
- Account doesn't have edit access to spreadsheet

**Solution:**
1. Go to n8n Settings → Credentials
2. Find your Google Sheets OAuth2 credential
3. Click "Test" or "Reconnect"
4. Re-authorize if needed
5. Verify the Google account has edit access to the spreadsheet

### Issue: Data Table Filter Not Working

**Symptoms**: Workflow processes disabled packages or misses enabled ones.

**Possible Causes:**
- Filter syntax incorrect
- `is_enabled` column has mixed true/false/null values

**Solution:**
1. Check filter syntax: `is_enabled = true` (or whatever your n8n version uses)
2. Verify data table has no rows with null `is_enabled`
3. Try removing filter temporarily to see all rows
4. Check n8n docs for correct filter syntax

### Issue: Duplicate Rows in Google Sheets

**Symptoms**: Same data written multiple times for same month.

**Possible Causes:**
- Workflow ran multiple times (manual + scheduled)
- Workflow didn't complete first time, ran again

**Solution:**
1. **Prevent**: Change Google Sheets operation from "Append" to "Update"
2. **Prevent**: Add IF condition to check if month already exists
3. **Fix**: Manually delete duplicate rows from Google Sheets
4. **Monitor**: Check execution logs to see why it ran multiple times

### Issue: Loop Doesn't Process All Packages

**Symptoms**: Only first package processed, others skipped.

**Possible Causes:**
- Loop back connection missing or incorrect
- "More Items to Process?" IF condition wrong

**Solution:**
1. Check workflow connections - ensure IF node "true" path goes back to Split in Batches
2. Verify IF condition uses `$('Split in Batches').context.noItemsLeft`
3. Test with execution logs to see where it stops

### Issue: Can't Find Workflow in Execute Workflow Dropdown

**Symptoms**: Task 3 sub-workflow doesn't appear in dropdown.

**Possible Causes:**
- Task 3 not created yet
- Task 3 not saved
- Task 3 not active
- Wrong workflow type (must be sub-workflow with Execute Workflow Trigger)

**Solution:**
1. Go to Workflows list, find "PyPI Stats Sub-Workflow"
2. Open it and verify it has "Execute Workflow Trigger" as first node
3. Make sure it's saved (no unsaved changes indicator)
4. Make sure it's active (toggle on)
5. Refresh Task 4 workflow and try dropdown again

## Maintenance

### Monthly Review Checklist

After each scheduled run (1st of month):

- [ ] Go to n8n → Executions
- [ ] Find the most recent run of this workflow
- [ ] Verify status is "Success" (green)
- [ ] Check execution time (should be <2 minutes normally)
- [ ] Open Google Sheets and verify new rows added
- [ ] Verify data looks reasonable:
  - Downloads > 0
  - Unique downloads < total downloads
  - Date is previous month
  - No null values
- [ ] Compare with previous month for unusual spikes/drops

### Adding New Packages

**Scenario**: Couchbase releases a new Python package.

**Steps**:
1. Add row to data table:
   - `package_name`: new package name (verify it exists on PyPI first!)
   - `tab_name`: descriptive tab name (e.g., "python-newpackage")
   - `is_enabled`: ✓ checked
2. Create matching tab in Google Sheets with column headers
3. Test manually:
   - Run workflow with manual trigger
   - Check new tab has data
   - Verify numbers look correct
4. If test passes, leave enabled for next scheduled run

### Temporarily Disabling Packages

**Scenario**: Need to stop tracking a package temporarily.

**Steps**:
1. Open data table `python_packages`
2. Find the package row
3. **Uncheck** `is_enabled` checkbox
4. Save
5. Next run will skip this package
6. Historical data in Google Sheets remains

**To re-enable**: Check the `is_enabled` box again.

### Updating Spreadsheet or Tabs

**Scenario**: Need to use different Google Sheets or rename tabs.

**To change spreadsheet**:
1. Open workflow
2. Click Google Sheets node
3. Update Document ID to new spreadsheet
4. Save workflow

**To rename tabs**:
1. Update `tab_name` in data table for affected packages
2. Rename tabs in Google Sheets to match
3. No workflow changes needed (uses dynamic tab name)

### Monitoring for Failures

**Set up notifications**:
1. In n8n, go to Settings → Workflows Settings
2. Enable "Error Workflow" (if available)
3. Create a simple workflow that sends email/Slack when errors occur
4. Link it as the error handler for this workflow

**Or manually check**:
- Review Executions tab weekly
- Look for red (failed) executions
- Investigate any failures immediately

## Future Enhancements

### 1. Add Error Notifications

**Why**: Get alerted immediately when workflow fails.

**How**:
- Add an error workflow or catch node
- Send email/Slack message on failure
- Include error details and affected package

### 2. Add Data Validation

**Why**: Catch anomalies in downloaded data.

**How**:
- Add IF conditions after sub-workflow call
- Check: `downloads > 0`, `unique_downloads <= downloads`, `no nulls`
- If validation fails, send alert instead of writing to sheets

### 3. Add Historical Comparison

**Why**: Detect unusual changes month-over-month.

**How**:
- After writing new data, read previous month's data from sheets
- Calculate % change
- Alert if change > 50% (might indicate data issue)

### 4. Add Retry Logic

**Why**: Handle temporary failures from pypistats.org.

**How**:
- Wrap sub-workflow call in try-catch
- If fails, wait 30 seconds and retry
- Retry up to 3 times before giving up

### 5. Create Dashboard

**Why**: Visualize trends over time.

**How**:
- In Google Sheets, create charts:
  - Line chart: Downloads over time per package
  - Bar chart: Month-over-month growth %
  - Pie chart: Download distribution across packages
- Link to dashboard from README

## Resources

- [n8n Data Tables Documentation](https://docs.n8n.io/data/)
- [n8n Loop Over Items](https://docs.n8n.io/workflows/loops/)
- [n8n Execute Workflow Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow/)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [PyPI Stats](https://pypistats.org/)
- [Cron Expression Generator](https://crontab.guru/)
- Task 3: PyPI Stats Sub-Workflow (dependency)

## Export & Version Control

### Exporting Workflow

After making changes:

1. Open workflow in n8n
2. Click **⋮** (three dots menu) → **Download**
3. Save as `package-stats-aggregator.json` in this directory (overwrite existing)
4. Commit to git:
   ```bash
   git add workflows/4-package-stats-aggregator/
   git commit -m "Update package aggregator workflow"
   git push
   ```

### Exporting Data Table

Unfortunately, n8n doesn't provide direct export of data tables. Document changes:

1. When you modify the data table structure, update `data-table-schema.json`
2. When you add packages, document in this README
3. Commit documentation changes to git

### Version History

Track major changes in git commits:
- Workflow logic changes → Update JSON + README
- Node configuration updates → Update JSON
- New packages added → Document in README
- Documentation improvements → Update README only

---

**Last Updated**: January 2025
**Maintainer**: Couchbase DevEx Team
**Questions**: See [QUICK-REFERENCE.md](../../docs/QUICK-REFERENCE.md) or reach out to the team
