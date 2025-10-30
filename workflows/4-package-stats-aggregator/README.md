# Task 4: Python Package Stats Aggregator (DA-1266)

## Overview

This is the **main workflow** that orchestrates monthly collection of Python package download statistics. It reads a list of packages from n8n Data Tables, calls the PyPI Stats Sub-Workflow (Task 3) for each enabled package, and stores results in Google Sheets.

**Purpose**: Automate monthly tracking of Couchbase Python package downloads from PyPI

**Schedule**: Runs automatically on the 1st of each month at 10:00 AM UTC

## Workflow Architecture

This workflow demonstrates the **orchestrator pattern**:
- Main workflow controls the flow
- Reads configuration from data table
- Calls sub-workflow multiple times (once per package)
- Aggregates and stores results

```
┌─────────────────────────────────────────────┐
│  Package Stats Aggregator (This Workflow)  │
│  ┌───────────────────────────────────────┐ │
│  │ 1. Trigger (Manual/Schedule)          │ │
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │ 2. Read Data Table                    │ │
│  │    - couchbase                        │ │
│  │    - langchain-couchbase              │ │
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │ 3. Loop Each Package (if enabled)     │ │
│  │    │                                  │ │
│  │    ├─> Call PyPI Sub-Workflow ────────┼─┼─> Task 3
│  │    │   (package_name, date)           │ │   Sub-Workflow
│  │    │                                  │ │
│  │    └─> Receive (downloads, unique)   <┼─┘
│  └───────────────────────────────────────┘ │
│  ┌───────────────────────────────────────┐ │
│  │ 4. Write to Google Sheets             │ │
│  │    Tab: python-couchbase              │ │
│  │    Tab: python-langchain              │ │
│  └───────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

## Data Table Configuration

### n8n Data Tables

n8n recently introduced **Native Data Tables** - built-in database storage for configuration and data.

**Why use Data Tables?**
- Store configuration without external database
- Easy to update package list via n8n UI
- Enable/disable packages without editing workflow
- Version controlled within n8n

### Create Data Table

1. Open n8n: <https://kaustavcouchbase.app.n8n.cloud>
2. Go to **Data** → **Tables** (or similar menu)
3. Create new table: `python_packages`

### Table Schema

| Column Name | Type | Description | Example |
|-------------|------|-------------|---------|
| `package_name` | Text | Python package name on PyPI | `couchbase` |
| `tab_name` | Text | Google Sheets tab for this package | `python-couchbase` |
| `is_enabled` | Boolean | Whether to track this package | `true` |

### Initial Data

Add these rows to the data table:

**Row 1**:
- `package_name`: `couchbase`
- `tab_name`: `python-couchbase`
- `is_enabled`: `true` (checked)

**Row 2**:
- `package_name`: `langchain-couchbase`
- `tab_name`: `python-langchain`
- `is_enabled`: `true` (checked)

### Adding More Packages

To track additional packages in the future:

1. Open the `python_packages` data table
2. Click **Add Row**
3. Fill in package details:
   - Package name (must exist on PyPI)
   - Tab name (will be created in Google Sheets if doesn't exist)
   - Is enabled (check to track, uncheck to skip)
4. Save
5. Next workflow run will include the new package

## Workflow Structure

### Nodes

1. **Manual Trigger** - For testing and ad-hoc runs

2. **Schedule Trigger** - Automatic monthly execution
   - Cron: `0 10 1 * *` (1st of month, 10:00 AM UTC)

3. **Read from Data Table** - Load package list
   - Table: `python_packages`
   - Filter: `is_enabled = true`

4. **Loop Over Items** - Process each package
   - Use **Split in Batches** node or **Loop Over Items** node

5. **Execute Workflow (Sub-Workflow)** - Call PyPI Stats Sub-Workflow
   - Workflow: PyPI Stats Sub-Workflow (Task 3)
   - Input: `package_name`, `date` (current month in yyyy-MM format)

6. **Code Node (Optional)** - Format data for Google Sheets
   - Calculate date string
   - Prepare row data

7. **Google Sheets (Append)** - Write results to spreadsheet
   - Spreadsheet ID: `1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8`
   - Sheet Name: `{{ $json.tab_name }}` (dynamic based on data table)

8. **Aggregate Results** - Collect all outputs

### Data Flow

```javascript
// Input from Data Table
{
  "package_name": "couchbase",
  "tab_name": "python-couchbase",
  "is_enabled": true
}

// After calling sub-workflow
{
  "package_name": "couchbase",
  "tab_name": "python-couchbase",
  "downloads": 45123,
  "unique_downloads": 38456,
  "date": "2025-01"
}

// Write to Google Sheets (python-couchbase tab)
Date       | Package Name | Downloads | Unique Downloads
2025-01    | couchbase    | 45123     | 38456
```

## Setup Instructions

### Step 1: Create Data Table

1. In n8n, navigate to **Data** → **Tables**
2. Click **Create Table**
3. Name: `python_packages`
4. Add columns:
   - `package_name` (Text)
   - `tab_name` (Text)
   - `is_enabled` (Boolean)
5. Add initial rows (couchbase, langchain-couchbase)
6. Save

### Step 2: Create Workflow

1. Click **Workflows** → **Add Workflow**
2. Name: "Python Package Stats Aggregator"

### Step 3: Configure Trigger Nodes

**Manual Trigger**:
- Add **Manual Trigger** node (for testing)

**Schedule Trigger**:
- Add **Schedule Trigger** node
- Mode: Custom
- Cron Expression: `0 10 1 * *`
- Timezone: UTC
- Description: "Run on 1st of month at 10:00 AM UTC"

### Step 4: Add Data Table Node

1. Add **Data Table** node (or equivalent read node)
2. Operation: **Read**
3. Table: Select `python_packages`
4. Filter: `is_enabled = true` (only enabled packages)

### Step 5: Add Loop Node

1. Add **Split in Batches** node or **Loop Over Items** node
2. Batch Size: 1 (process one package at a time)
3. This ensures sub-workflow is called once per package

### Step 6: Add Execute Workflow Node

1. Add **Execute Workflow** node
2. Source: **Database** (select workflow by name)
3. Workflow: Select "PyPI Stats Sub-Workflow" (Task 3)
4. Parameters:
   - `package_name`: `{{ $json.package_name }}`
   - `date`: `{{ $now.format('YYYY-MM') }}` or use Code node to calculate previous month

### Step 7: Add Code Node (Format Date)

Optional but recommended to calculate previous month's date:

```javascript
// Calculate previous month (since we run on 1st of month)
const now = new Date();
const lastMonth = new Date(now.getFullYear(), now.getMonth() - 1, 1);
const dateStr = lastMonth.toISOString().slice(0, 7); // "2025-01"

return {
  json: {
    ...item.json,
    date: dateStr
  }
};
```

### Step 8: Add Google Sheets Node

1. Add **Google Sheets** node
2. Credential: Select Google Sheets OAuth2/Service Account
3. Operation: **Append**
4. Spreadsheet: `1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8`
5. Sheet Name: `{{ $json.tab_name }}` (dynamic from data table)
6. Columns:
   - Date: `{{ $json.date }}`
   - Package Name: `{{ $json.package_name }}`
   - Downloads: `{{ $json.downloads }}`
   - Unique Downloads: `{{ $json.unique_downloads }}`

### Step 9: Test Workflow

1. Click **Execute Workflow** (manual trigger)
2. Verify:
   - Data table read successfully
   - Sub-workflow called for each enabled package
   - Results written to correct Google Sheets tabs
3. Check Google Sheets for new rows

### Step 10: Activate

1. Toggle **Active** to enable the workflow
2. Schedule will run automatically on 1st of each month

## Google Sheets Setup

### Spreadsheet

**URL**: <https://docs.google.com/spreadsheets/d/1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8/>

### Tabs

Create these tabs in the spreadsheet:

#### python-couchbase

| Date | Package Name | Downloads | Unique Downloads |
|------|--------------|-----------|------------------|
| 2025-01 | couchbase | 45123 | 38456 |
| 2024-12 | couchbase | 43890 | 37210 |

#### python-langchain

| Date | Package Name | Downloads | Unique Downloads |
|------|--------------|-----------|------------------|
| 2025-01 | langchain-couchbase | 8234 | 6890 |
| 2024-12 | langchain-couchbase | 7856 | 6234 |

### Permissions

**For Service Account**:
1. Share spreadsheet with service account email
2. Give "Editor" permission

**For OAuth2**:
1. Ensure OAuth app has spreadsheet edit scope
2. Authorize via n8n credential setup

## Testing

### Test Checklist

Before enabling the schedule, verify:

- [ ] Data table exists with correct schema
- [ ] Initial packages added (couchbase, langchain-couchbase)
- [ ] Both packages marked as enabled
- [ ] Sub-workflow (Task 3) is active and working
- [ ] Google Sheets credential configured
- [ ] Spreadsheet tabs exist (python-couchbase, python-langchain)
- [ ] Manual execution succeeds
- [ ] Data written to correct tabs in Google Sheets
- [ ] Values look reasonable (not null)

### Test with One Package

To test safely:

1. Temporarily disable one package in data table
2. Run workflow manually
3. Verify only enabled package processed
4. Re-enable the package after testing

## Scheduling Logic

### Run on 1st of Month

**Why**: This allows collecting complete statistics for the previous month

**Timing**: 10:00 AM UTC on the 1st

**Date Calculation**: Workflow should request stats for **previous month** (the month that just ended)

Example:
- Run Date: February 1, 2025
- Data Requested: January 2025 (`2025-01`)

### Cron Expression

`0 10 1 * *`

Breakdown:
- `0` - Minute 0
- `10` - Hour 10 (10 AM)
- `1` - Day 1 (first day of month)
- `*` - Every month
- `*` - Every day of week

## Troubleshooting

### Issue: Sub-workflow not found

**Cause**: PyPI Stats Sub-Workflow not active or wrong workflow ID

**Solution**:
1. Verify Task 3 sub-workflow is active
2. Check workflow name matches exactly
3. Try selecting by ID instead of name

### Issue: Data table empty

**Cause**: Incorrect filter or table name

**Solution**:
1. Verify table name: `python_packages`
2. Check filter syntax: `is_enabled = true`
3. Manually verify data exists in table

### Issue: Wrong Google Sheets tab

**Cause**: Tab name mismatch between data table and Google Sheets

**Solution**:
1. Ensure tab names match exactly (case-sensitive)
2. Create missing tabs in Google Sheets
3. Update data table if tab name changed

### Issue: No data written to sheets

**Cause**: Sub-workflow returned null or error

**Solution**:
1. Check sub-workflow execution logs
2. Verify package names are correct on PyPI
3. Ensure date is within 3-month window
4. Test sub-workflow independently

### Issue: Duplicate rows in sheets

**Cause**: Workflow ran multiple times for same month

**Solution**:
1. Use **Update** instead of **Append** in Google Sheets node
2. Add conditional logic to check if data for month already exists
3. Manually remove duplicates if needed

## Maintenance

### Monthly Review

After each automated run:

- [ ] Check execution logs in n8n
- [ ] Verify all enabled packages processed
- [ ] Review Google Sheets for new data
- [ ] Ensure no errors or null values
- [ ] Compare with previous month for anomalies

### Adding New Packages

When Couchbase releases new Python packages:

1. Add row to `python_packages` data table
2. Create corresponding tab in Google Sheets
3. Set `is_enabled` to `true`
4. Test with manual trigger before waiting for scheduled run

### Disabling Packages

To stop tracking a package:

1. Open `python_packages` data table
2. Uncheck `is_enabled` for that package
3. Workflow will skip it on next run
4. Keep historical data in Google Sheets tab

## Future Enhancements

### Error Handling

```javascript
// Wrap sub-workflow call in try-catch
try {
  const result = await executeSubWorkflow();
  return result;
} catch (error) {
  // Log error, send notification
  return { error: error.message, package_name: item.json.package_name };
}
```

### Email Notifications

Add **Send Email** node:
- Trigger: On workflow completion
- Content: Summary of packages processed, any errors
- Recipients: DevEx team

### Slack Notifications

Add **Slack** node:
- Message: "Monthly PyPI stats collected: couchbase (45K downloads), langchain-couchbase (8K downloads)"
- Channel: #devex-metrics

### Historical Charts

Create visualizations in Google Sheets:
- Line chart: Downloads over time per package
- Bar chart: Month-over-month growth
- Table: Top performing packages

### Data Validation

Add validation node to check:
- Downloads > 0
- Unique downloads ≤ downloads
- Date format correct
- No null values

## Resources

- [n8n Data Tables Documentation](https://docs.n8n.io/data/)
- [n8n Loop Over Items](https://docs.n8n.io/workflows/loops/)
- [n8n Execute Workflow Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow/)
- [Google Sheets API](https://developers.google.com/sheets/api)
- [PyPI Stats](https://pypistats.org/)
- Task 3: PyPI Stats Sub-Workflow (dependency)

## Export Workflow

After configuring the workflow:

1. Open the workflow in n8n
2. Click **⋮** (three dots) → **Download**
3. Save as `package-stats-aggregator.json` in this directory
4. Also export the data table schema if possible
5. Commit to git

## Data Table Schema Export

Document the data table structure for future reference:

```json
{
  "table_name": "python_packages",
  "columns": [
    { "name": "package_name", "type": "text" },
    { "name": "tab_name", "type": "text" },
    { "name": "is_enabled", "type": "boolean" }
  ],
  "initial_data": [
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
}
```

Save this as `data-table-schema.json` in this directory.
