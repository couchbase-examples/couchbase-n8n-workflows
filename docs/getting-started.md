# Getting Started with Couchbase n8n Workflows

Welcome to the Couchbase DevEx team's n8n workflows repository! This guide will help you get up and running with our automated statistics collection workflows.

## What is n8n?

n8n is a workflow automation platform (like Zapier, but self-hosted) that allows you to connect different services and automate tasks. We use it to collect statistics about Couchbase's developer tools and libraries.

## Repository Overview

This repository contains workflows that:

- **Track JetBrains plugin statistics** - Monthly download counts for the Couchbase IntelliJ plugin
- **Track Python package statistics** - Download metrics from PyPI for our Python SDKs
- **Store historical data** - All metrics are saved to Google Sheets for analysis

All workflows run automatically on the 1st of each month, with manual trigger options for testing.

## Prerequisites

1. **Access to n8n Cloud**: https://kaustavcouchbase.app.n8n.cloud
2. **Credentials configured in n8n**:
   - JetBrains Account (email/password)
   - Google Sheets API access
   - n8n API Key for Claude Code integration
3. **Claude Code with MCP** (optional, for AI-assisted workflow management)

## Getting Your n8n API Key

To enable Claude Code integration with n8n:

1. Log into n8n Cloud: https://kaustavcouchbase.app.n8n.cloud
2. Click your profile icon (top right) → **Settings**
3. Navigate to **API** section
4. Click **Create an API Key** and copy it
5. Add to your shell environment:
   ```bash
   export N8N_API_KEY="your-api-key-here"
   ```
6. Make it permanent by adding to `~/.zshrc` or `~/.bashrc`

## Repository Structure

```
couchbase-n8n-workflows/
├── .mcp.json                 # Claude Code MCP configuration
├── .gitignore                # Git ignore patterns
├── tasks/                    # Original task specifications (DA-1261 to DA-1266)
├── workflows/                # n8n workflow JSON exports & documentation
│   ├── 1-deploy-n8n-cluster/
│   ├── 2-jetbrains-stats/
│   ├── 3-pypistats-subworkflow/
│   └── 4-package-stats-aggregator/
└── docs/
    └── getting-started.md    # This file
```

## Importing Workflows into n8n

To import a workflow from this repository:

1. Navigate to the workflow folder (e.g., `workflows/2-jetbrains-stats/`)
2. Copy the JSON file content (e.g., `jetbrains-stats.json`)
3. In n8n Cloud, click **Workflows** → **Import from File** or **Import from URL**
4. Paste the JSON content
5. Click **Import**
6. Configure required credentials (see workflow README)
7. Test with the manual trigger node before enabling the schedule

## Exporting Workflows from n8n

After making changes to a workflow in n8n:

1. Open the workflow in n8n
2. Click the **⋮** (three dots) menu → **Download**
3. Save the JSON file to the appropriate workflow folder
4. Update the workflow's README.md with any changes
5. Commit both files to git:
   ```bash
   git add workflows/2-jetbrains-stats/
   git commit -m "Update JetBrains stats workflow"
   git push
   ```

## Understanding Workflow Types

### Main Workflows

Workflows that can be triggered manually or on a schedule. These are the entry points.

**Examples**: JetBrains Stats (Task 2), Package Stats Aggregator (Task 4)

### Sub-Workflows

Reusable workflows called by main workflows (like functions in programming). They accept inputs and return outputs.

**Example**: PyPI Stats Sub-workflow (Task 3) - takes a package name and returns download stats

## Testing Workflows

Before scheduling a workflow to run automatically:

1. **Use Manual Trigger**: Every workflow has a manual trigger node for testing
2. **Check Credentials**: Ensure all API keys and credentials are configured
3. **Verify Data Tables**: For Task 4, ensure the n8n Data Table is populated
4. **Test in Stages**: For complex workflows with multiple steps, test each node individually using n8n's "Execute Node" feature
5. **Check Output**: Verify data is correctly written to Google Sheets

## Puppeteer Setup

Some workflows (Task 2) use Puppeteer for browser automation. This requires:

- **n8n Docker Image**: Must use `n8nio/n8n:latest` or puppeteer-enabled image
- **Browser Context**: Workflows create a headless browser to navigate websites
- **Screenshots**: For debugging, workflows can capture screenshots (saved as base64)

## Common Issues & Troubleshooting

### Workflow fails with "Unauthorized"

- Check that credentials are properly configured in n8n Settings → Credentials
- Verify API keys haven't expired

### Puppeteer script times out

- Increase the timeout value in the Code node
- Check if the target website structure has changed
- Review base64 screenshot output for debugging

### Google Sheets not updating

- Verify the Google Sheets credential has edit permissions
- Check that the spreadsheet ID in the workflow matches the target sheet
- Ensure the correct tab name is specified

### Sub-workflow not found

- Make sure the sub-workflow is active in n8n
- Verify the sub-workflow ID matches in the calling workflow
- Check that the sub-workflow has been properly saved

## Data Storage

### Google Sheets

All statistics are stored in: https://docs.google.com/spreadsheets/d/1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8/

**Tabs**:

- `jetbrains` - JetBrains plugin download stats
- `python-couchbase` - couchbase package stats from PyPI
- `python-langchain` - langchain-couchbase package stats from PyPI

### n8n Data Tables

For Task 4, a data table stores the list of packages to track:

| Column Name  | Type    | Description                             |
| ------------ | ------- | --------------------------------------- |
| package_name | Text    | Python package name (e.g., "couchbase") |
| tab_name     | Text    | Google Sheets tab to save results       |
| is_enabled   | Boolean | Whether to track this package           |

## n8n Best Practices

1. **Name nodes clearly**: Use descriptive names like "Get JetBrains Stats" instead of "HTTP Request 1"
2. **Add notes**: Use the note feature to document complex logic
3. **Use error workflows**: Configure error handling to get notified when workflows fail
4. **Version control**: Export workflows after significant changes
5. **Test before scheduling**: Always test manually before enabling scheduled triggers
6. **Keep credentials secure**: Never commit API keys to git - use n8n's credential management

## Scheduled Execution

Workflows run on the 1st of each month at scheduled times:

- **JetBrains Stats**: 1st of month, 9:00 AM UTC
- **Package Stats Aggregator**: 1st of month, 10:00 AM UTC

To modify schedules:

1. Open the workflow in n8n
2. Click the **Schedule Trigger** node
3. Adjust the cron expression or use the visual editor
4. Save the workflow

## Getting Help

- **n8n Documentation**: https://docs.n8n.io/
- **n8n Community Forum**: https://community.n8n.io/
- **Team**: Reach out to @Dima, @Kaustav, @Niranjan, @Vishal, or @Priya
- **Task Details**: See individual workflow READMEs in the `workflows/` folder

## Next Steps

1. Review the workflow READMEs in the `workflows/` directory
2. Import one workflow to understand the structure
3. Run a test execution with the manual trigger
4. Review the Google Sheets output
5. Make modifications as needed and export back to this repo

Happy automating!
