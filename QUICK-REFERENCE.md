# Quick Reference Guide

## Repository Overview

This repository contains n8n workflows for automating monthly statistics collection for Couchbase developer tools and libraries.

**n8n Instance**: <https://kaustavcouchbase.app.n8n.cloud>
**Google Sheets**: <https://docs.google.com/spreadsheets/d/1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8/>

## The 4 Tasks

| Task | Type | What It Does | JSON File |
|------|------|--------------|-----------|
| **Task 1** | Setup | Deploy n8n cluster, add users, configure Puppeteer | ❌ No workflow (it's infrastructure setup) |
| **Task 2** | Workflow | Scrape JetBrains plugin download stats with Puppeteer | ✅ `jetbrains-stats.json` |
| **Task 3** | Sub-Workflow | Fetch PyPI package download stats (reusable) | ✅ `pypistats-subworkflow.json` |
| **Task 4** | Workflow | Orchestrate Task 3 for multiple Python packages | ✅ `package-stats-aggregator.json` |

## Why Task 1 Has No JSON

**Task 1 is not a workflow** - it's the infrastructure setup:
- Deploying the n8n Cloud instance itself
- Installing Puppeteer support
- Creating user accounts for the DevEx team (Niranjan, Vishal, Priya)

It's the **foundation** that allows the other workflows to run. Think of it as "building the factory" before creating the assembly lines.

## Quick Setup Steps

### 1. Set Up Google Sheets Credential (Required for Tasks 2 & 4)

```
n8n → Settings → Credentials → Add Credential
Search: "Google Sheets OAuth2 API"
Authorize your Google account
```

### 2. Create Data Table (Required for Task 4)

```
n8n → Data → Tables → Create Table
Name: python_packages
Columns:
  - package_name (text)
  - tab_name (text)
  - is_enabled (boolean)
Initial rows:
  - couchbase, python-couchbase, ✓
  - langchain-couchbase, python-langchain, ✓
```

### 3. Import Workflows (In This Order!)

```bash
# Order matters because Task 4 calls Task 3

1. Import Task 3 first:
   workflows/3-pypistats-subworkflow/pypistats-subworkflow.json
   → Note the Workflow ID after import

2. Import Task 4:
   workflows/4-package-stats-aggregator/package-stats-aggregator.json
   → Edit "Call PyPI Stats Sub-Workflow" node
   → Select Task 3 by name or ID

3. Import Task 2:
   workflows/2-jetbrains-stats/jetbrains-stats.json
   → Edit Code node: Replace EMAIL and PASSWORD
```

### 4. Get n8n API Key (Optional - For Claude Code Integration)

```bash
# In n8n
Profile → Settings → API → Create API Key

# In terminal
export N8N_API_KEY="your-key-here"
echo 'export N8N_API_KEY="your-key-here"' >> ~/.zshrc
```

## File Structure

```
couchbase-n8n-workflows/
├── QUICK-REFERENCE.md          # This file
├── .mcp.json                   # Claude Code integration config
├── .gitignore                  # Git ignore patterns
├── docs/
│   └── getting-started.md      # Detailed onboarding guide
├── tasks/                      # Original task specifications
│   ├── all.txt                 # Task overview
│   ├── 1.txt → 4.txt          # Individual task details
└── workflows/                  # Workflow documentation & JSON exports
    ├── 1-deploy-n8n-cluster/
    │   └── README.md           # Infrastructure setup guide
    ├── 2-jetbrains-stats/
    │   ├── README.md           # Workflow documentation
    │   └── jetbrains-stats.json # Importable workflow
    ├── 3-pypistats-subworkflow/
    │   ├── README.md
    │   └── pypistats-subworkflow.json
    └── 4-package-stats-aggregator/
        ├── README.md
        ├── package-stats-aggregator.json
        └── data-table-schema.json # Data table structure
```

## Credentials Needed

| Credential | Used By | Type | Setup |
|------------|---------|------|-------|
| **Google Sheets OAuth2** | Tasks 2, 4 | OAuth2 | n8n Settings → Credentials |
| **JetBrains Account** | Task 2 | Email/Password | Hardcoded in Code node (or use Custom Auth) |
| **n8n API Key** | Claude Code | API Key | Profile → Settings → API |

## Workflow Schedules

| Workflow | Schedule | Cron Expression | Timezone |
|----------|----------|-----------------|----------|
| Task 2 - JetBrains Stats | 1st of month, 9:00 AM | `0 9 1 * *` | UTC |
| Task 4 - Package Stats | 1st of month, 10:00 AM | `0 10 1 * *` | UTC |

**Why these times?** Running on the 1st allows collecting complete statistics for the previous month.

## Testing Workflows

### Test Task 3 (PyPI Sub-Workflow)

```
1. Open Task 3 workflow in n8n
2. Click "Execute Workflow" (it has no manual trigger, it's callable)
3. Actually, create a test workflow:
   - Manual Trigger
   - Execute Workflow node → Select Task 3
   - Pass test data: { package_name: "couchbase", date: "2025-01" }
4. Verify output has downloads and unique_downloads
```

### Test Task 4 (Package Aggregator)

```
1. Ensure data table exists with enabled packages
2. Click "Execute Workflow" (manual trigger)
3. Watch execution logs - should loop through packages
4. Verify data written to Google Sheets tabs
```

### Test Task 2 (JetBrains Stats)

```
1. Update EMAIL and PASSWORD in Code node
2. Click "Execute Workflow"
3. Check execution time (may take 2-3 minutes due to Puppeteer)
4. If fails, check screenshot_base64 output for debugging
```

## Common Issues

### Issue: Google Sheets permission denied

**Solution**: Ensure OAuth2 credential is authorized and spreadsheet is shared with your Google account

### Issue: Data table not found (Task 4)

**Solution**: Create `python_packages` table in n8n Data section (see setup steps)

### Issue: Sub-workflow not found (Task 4)

**Solution**: Ensure Task 3 is imported and active, then reference it by name in Task 4

### Issue: Cookie banner not dismissed (Task 2)

**Known issue**: The workflow includes extensive cookie handling logic but may still fail. Check the `screenshot_base64` field in output to see actual page state. May require manual debugging.

### Issue: PyPI stats parsing fails (Task 3)

**Solution**: Visit pypistats.org manually to inspect HTML structure, update parsing logic in Code node

## Important Dates

- **Scheduled Runs**: 1st of each month
- **Data Collected For**: Previous month (e.g., run on Feb 1 collects Jan data)
- **PyPI Data Availability**: Limited to past 3 months

## Google Sheets Structure

### Tab: jetbrains

| Year-Month | Total Downloads | Downloads This Month | Date Collected |
|------------|-----------------|----------------------|----------------|
| 2025-01 | 15234 | 342 | 2025-02-01 |

### Tab: python-couchbase

| Date | Package Name | Downloads | Unique Downloads |
|------|--------------|-----------|------------------|
| 2025-01 | couchbase | 45123 | 38456 |

### Tab: python-langchain

| Date | Package Name | Downloads | Unique Downloads |
|------|--------------|-----------|------------------|
| 2025-01 | langchain-couchbase | 8234 | 6890 |

## Adding New Packages

To track additional Python packages:

```
1. Open n8n → Data → Tables → python_packages
2. Click "Add Row"
3. Fill in:
   - package_name: New package name on PyPI
   - tab_name: Google Sheets tab to use
   - is_enabled: ✓ (check)
4. Save
5. Create corresponding tab in Google Sheets (or let workflow create it)
6. Next scheduled run will include the package
```

## Git Workflow

```bash
# After making changes in n8n, export and commit

# Export workflow from n8n
n8n → Workflow → Menu (⋮) → Download

# Save to appropriate folder
mv ~/Downloads/workflow.json workflows/2-jetbrains-stats/jetbrains-stats.json

# Commit changes
git add .
git commit -m "Update JetBrains stats workflow"
git push
```

## Useful Commands

```bash
# View repository structure
tree -L 3 -I 'node_modules|.git'

# Check git status
git status

# Set n8n API key
export N8N_API_KEY="your-key"

# View workflow logs (if running locally)
# n8n Cloud has built-in execution logs in the UI
```

## Documentation Links

- **Detailed Setup**: [`docs/getting-started.md`](docs/getting-started.md)
- **Task 1 (Deploy n8n)**: [`workflows/1-deploy-n8n-cluster/README.md`](workflows/1-deploy-n8n-cluster/README.md)
- **Task 2 (JetBrains)**: [`workflows/2-jetbrains-stats/README.md`](workflows/2-jetbrains-stats/README.md)
- **Task 3 (PyPI)**: [`workflows/3-pypistats-subworkflow/README.md`](workflows/3-pypistats-subworkflow/README.md)
- **Task 4 (Aggregator)**: [`workflows/4-package-stats-aggregator/README.md`](workflows/4-package-stats-aggregator/README.md)

## External Resources

- **n8n Documentation**: <https://docs.n8n.io/>
- **n8n Community**: <https://community.n8n.io/>
- **Puppeteer Docs**: <https://pptr.dev/>
- **PyPI Stats**: <https://pypistats.org/>
- **JetBrains Marketplace**: <https://plugins.jetbrains.com/>

## Team Contacts

- **Dima Checketkin** - DevEx Team Lead (Task 1 assignee)
- **Kaustav Ghosh** - DevEx Team (Tasks 2, 3, 4 assignee)
- **Niranjan, Vishal, Priya** - DevEx Team Members

## Quick Troubleshooting

```
Problem: Workflow not running on schedule
→ Check workflow is Active (toggle in n8n)
→ Verify cron expression is correct
→ Check n8n Cloud execution limits

Problem: No data in Google Sheets
→ Verify credential is authorized
→ Check workflow execution logs for errors
→ Ensure spreadsheet tabs exist

Problem: Sub-workflow returns null
→ Test sub-workflow independently
→ Check input parameters are passed correctly
→ Verify package name is correct on PyPI

Problem: Puppeteer timeout
→ Increase timeout values in Code node
→ Check screenshot_base64 for page state
→ Verify target website is accessible
```

## Next Steps

1. ✅ Set up Google Sheets credential
2. ✅ Create python_packages data table
3. ✅ Import workflows (Task 3 → Task 4 → Task 2)
4. ✅ Test each workflow manually
5. ✅ Activate schedules
6. 📅 Monitor first scheduled run on 1st of next month

---

**Last Updated**: January 2025
**Repository**: <https://github.com/couchbase/couchbase-n8n-workflows> (or your actual repo URL)
