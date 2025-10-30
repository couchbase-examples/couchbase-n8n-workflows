# Couchbase n8n Workflows

Automated statistics collection workflows for Couchbase developer tools and libraries using n8n workflow automation.

## Overview

This repository contains n8n workflows that automatically collect and track download statistics for Couchbase's developer tools:

- **JetBrains Plugin** - Monthly download counts for the Couchbase IntelliJ plugin
- **Python Packages** - Download metrics from PyPI for `couchbase` and `langchain-couchbase`

All metrics are automatically collected on the 1st of each month and stored in Google Sheets for historical analysis.

**n8n Instance**: <https://kaustavcouchbase.app.n8n.cloud>
**Google Sheets**: [View Statistics](https://docs.google.com/spreadsheets/d/1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8/)

## What is n8n?

n8n is a workflow automation platform (like Zapier, but self-hosted) that allows you to connect different services and automate tasks. We use it to collect statistics about Couchbase's developer tools without manual intervention.

## Repository Structure

```
couchbase-n8n-workflows/
├── README.md                 # This file
├── .mcp.json                 # Claude Code MCP configuration
├── .gitignore                # Git ignore patterns
├── tasks/                    # Original task specifications
├── docs/                     # Documentation
│   ├── getting-started.md    # Detailed setup guide
│   └── QUICK-REFERENCE.md    # Quick reference for common tasks
└── workflows/                # n8n workflow JSON exports & documentation
    ├── 1-deploy-n8n-cluster/
    │   └── README.md         # Infrastructure setup guide
    ├── 2-jetbrains-stats/
    │   ├── README.md         # Workflow documentation
    │   └── jetbrains-stats.json
    ├── 3-pypistats-subworkflow/
    │   ├── README.md
    │   └── pypistats-subworkflow.json
    └── 4-package-stats-aggregator/
        ├── README.md
        ├── package-stats-aggregator.json
        └── data-table-schema.json
```

## Quick Start

### Prerequisites

1. Access to n8n Cloud: <https://kaustavcouchbase.app.n8n.cloud>
2. Google account with access to the statistics spreadsheet
3. JetBrains account credentials

### Setup Steps

1. **Set up Google Sheets credential** (required for workflows 2 & 4)

   ```
   n8n → Settings → Credentials → Add Credential
   Search: "Google Sheets OAuth2 API"
   Authorize your Google account
   ```

2. **Create data table** (required for workflow 4)

   ```
   n8n → Data → Tables → Create Table
   Name: python_packages
   Columns: package_name (text), tab_name (text), is_enabled (boolean)
   Add rows:
     - couchbase, python-couchbase, ✓
     - langchain-couchbase, python-langchain, ✓
   ```

3. **Import workflows in order**

   - Import Task 3 first: `workflows/3-pypistats-subworkflow/pypistats-subworkflow.json`
   - Then Task 4: `workflows/4-package-stats-aggregator/package-stats-aggregator.json`
   - Finally Task 2: `workflows/2-jetbrains-stats/jetbrains-stats.json`

4. **Configure credentials in each workflow**

   - Update JetBrains email/password in Task 2
   - Select Google Sheets credential in Tasks 2 & 4
   - Link Task 4 to Task 3 sub-workflow

5. **Test workflows manually** before enabling schedules

For quick reference and troubleshooting, see [docs/QUICK-REFERENCE.md](docs/QUICK-REFERENCE.md).

## The 4 Tasks

| Task                                            | Type           | Description                                        | Schedule                   |
| ----------------------------------------------- | -------------- | -------------------------------------------------- | -------------------------- |
| [Task 1](workflows/1-deploy-n8n-cluster/)       | Infrastructure | n8n cluster deployment and setup                   | One-time setup             |
| [Task 2](workflows/2-jetbrains-stats/)          | Workflow       | Scrape JetBrains plugin stats using Puppeteer      | 1st of month, 9:00 AM UTC  |
| [Task 3](workflows/3-pypistats-subworkflow/)    | Sub-Workflow   | Fetch PyPI package download stats (reusable)       | Called by Task 4           |
| [Task 4](workflows/4-package-stats-aggregator/) | Workflow       | Orchestrate stats collection for multiple packages | 1st of month, 10:00 AM UTC |

## Workflow Types

### Main Workflows (Tasks 2 & 4)

Standalone workflows that can be triggered manually or on a schedule. These are the entry points for automation.

### Sub-Workflows (Task 3)

Reusable workflow components called by main workflows (like functions in programming). They accept inputs and return outputs, promoting code reuse.

## Data Storage

### Google Sheets

All statistics are stored in: <https://docs.google.com/spreadsheets/d/1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8/>

**Tabs:**

- `jetbrains` - JetBrains plugin download stats
- `python-couchbase` - couchbase package stats from PyPI
- `python-langchain` - langchain-couchbase package stats from PyPI

### n8n Data Tables

Task 4 uses an n8n data table (`python_packages`) to store the list of Python packages to track. This allows easy addition/removal of packages without editing the workflow.

## Importing & Exporting Workflows

### Import into n8n

1. Navigate to workflow folder (e.g., `workflows/2-jetbrains-stats/`)
2. In n8n Cloud, click **Workflows** → **Import from File**
3. Upload the `.json` file
4. Configure credentials as documented in the workflow README
5. Test with manual trigger before enabling schedule

### Export from n8n

After making changes:

1. Open workflow in n8n → **⋮** menu → **Download**
2. Save JSON file to appropriate workflow folder
3. Update workflow README if needed
4. Commit and push:
   ```bash
   git add workflows/
   git commit -m "Update workflow"
   git push
   ```

## Adding New Python Packages

To track additional packages from PyPI:

1. Open n8n → **Data** → **Tables** → `python_packages`
2. Click **Add Row**
3. Fill in:
   - `package_name`: Package name on PyPI
   - `tab_name`: Google Sheets tab to use
   - `is_enabled`: ✓ (checked)
4. Create corresponding tab in Google Sheets (or let workflow create it)
5. Next scheduled run will include the package

## Common Issues & Troubleshooting

### Workflow fails with "Unauthorized"

- Verify credentials are configured in n8n Settings → Credentials
- Check API keys haven't expired

### Puppeteer script times out (Task 2)

- Increase timeout value in Code node
- Check if target website structure changed
- Review base64 screenshot output for debugging

### Google Sheets not updating

- Verify Google Sheets credential has edit permissions
- Check spreadsheet ID matches target sheet
- Ensure tab names are correct

### Sub-workflow not found (Task 4)

- Ensure Task 3 sub-workflow is active
- Verify workflow ID/name in Task 4 matches Task 3
- Check sub-workflow saved properly

## Technology Stack

- **n8n** - Workflow automation platform
- **Puppeteer** - Headless browser automation for web scraping
- **Google Sheets API** - Data storage and historical tracking
- **pypistats.org** - PyPI download statistics source
- **JetBrains Marketplace** - Plugin statistics source

## Documentation

- **[Quick Reference](docs/QUICK-REFERENCE.md)** - Quick lookup and troubleshooting guide
- **[Workflow READMEs](workflows/)** - Individual workflow documentation with detailed setup instructions

## External Resources

- [n8n Documentation](https://docs.n8n.io/)
- [n8n Community Forum](https://community.n8n.io/)
- [Puppeteer Documentation](https://pptr.dev/)
- [PyPI Stats](https://pypistats.org/)
- [JetBrains Marketplace](https://plugins.jetbrains.com/)

## Team

- **Dima Checketkin** - DevEx Team Lead
- **Kaustav Ghosh** - DevEx Team
- **Niranjan, Vishal, Priya** - DevEx Team Members

## Getting Help

- Review workflow READMEs in the `workflows/` directory
- Check [troubleshooting section](docs/getting-started.md#common-issues--troubleshooting)
- Reach out to the DevEx team: @Dima, @Kaustav, @Niranjan, @Vishal, @Priya

## Contributing

When making changes to workflows:

1. Export updated workflow from n8n
2. Save JSON to appropriate folder
3. Update workflow README
4. Commit changes with descriptive message
5. Push to repository

## License

Copyright © 2025 Couchbase, Inc.

---

**Last Updated**: January 2025
**Repository**: <https://github.com/couchbase-examples/couchbase-n8n-workflows>
