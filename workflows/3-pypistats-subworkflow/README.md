# Task 3: PyPI Download Statistics Sub-Workflow (DA-1265)

## Overview

This is a **reusable sub-workflow** that fetches download statistics for Python packages from PyPI Stats (<https://pypistats.org>). It can be called by other workflows (like Task 4) to retrieve download metrics for multiple packages.

**Purpose**: Get monthly download counts for Python packages hosted on PyPI

## Sub-Workflow Concept

In n8n, a **sub-workflow** is like a function in programming:
- It accepts input parameters
- Performs operations
- Returns output values
- Can be called by multiple parent workflows

This makes code reusable and easier to maintain.

## Input Parameters

The sub-workflow expects these inputs:

| Parameter | Type | Format | Example | Description |
|-----------|------|--------|---------|-------------|
| `package_name` | String | - | `couchbase` | Python package name on PyPI |
| `date` | String | `yyyy-MM` | `2025-01` | Year-Month for statistics (optional) |

**Note**: PyPI Stats typically limits data to the **past 3 months**. Older dates may not return data.

## Output Values

The sub-workflow returns:

| Field | Type | Description |
|-------|------|-------------|
| `downloads` | Integer | Total downloads including mirrors (with_mirrors) |
| `unique_downloads` | Integer | Unique downloads without mirrors (without_mirrors) |
| `package_name` | String | Echo of input package name |
| `date` | String | Echo of input date |

## How It Works

### Flow

1. **Receive input** - Get `package_name` and `date` from calling workflow
2. **Construct URL** - Build `https://pypistats.org/packages/{package_name}`
3. **Fetch page** - HTTP GET request or web scraping
4. **Parse data** - Extract download statistics for specified month
5. **Return output** - Send `downloads` and `unique_downloads` back to caller

### Implementation Options

#### Option 1: HTTP Request (Preferred)

Check if pypistats.org provides an API or JSON endpoint:

```javascript
// Code Node
const packageName = $input.item.json.package_name;
const date = $input.item.json.date; // e.g., "2025-01"

// Try fetching JSON data if available
const url = `https://pypistats.org/api/packages/${packageName}/recent`;
// Parse response and extract metrics for specified month
```

**Advantages**: Fast, reliable, no HTML parsing needed

**Check**: Inspect <https://pypistats.org/packages/couchbase> network traffic for API endpoints

#### Option 2: Web Scraping

If no API exists, scrape the HTML page:

```javascript
// Code Node with jsdom or cheerio
const packageName = $input.item.json.package_name;
const url = `https://pypistats.org/packages/${packageName}`;

// Fetch HTML
// Parse table/chart data
// Extract downloads (with mirrors) and unique downloads (without mirrors)
```

**Note**: May require Puppeteer if the page uses JavaScript to render statistics

## Workflow Structure

### Nodes

1. **Execute Workflow Trigger** - Entry point when called by other workflows
   - Input: `package_name`, `date`

2. **HTTP Request Node** or **Code Node** - Fetch data from pypistats.org
   - Method: GET
   - URL: `https://pypistats.org/packages/{{ $json.package_name }}`

3. **Code Node** - Parse HTML/JSON and extract metrics
   - Parse downloads (with mirrors)
   - Parse unique downloads (without mirrors)

4. **Respond to Workflow** - Return output to calling workflow
   - Output: `{ downloads, unique_downloads, package_name, date }`

## Setup Instructions

### Step 1: Create Sub-Workflow

1. Open n8n: <https://kaustavcouchbase.app.n8n.cloud>
2. Click **Workflows** → **Add Workflow**
3. Name it: "PyPI Stats Sub-Workflow"

### Step 2: Configure Trigger

1. Add **Execute Workflow Trigger** node
2. This makes the workflow callable from other workflows
3. It will receive input data from the parent workflow

### Step 3: Add HTTP Request or Code Node

**Option A: HTTP Request Node**

1. Add **HTTP Request** node
2. Method: `GET`
3. URL: `https://pypistats.org/packages/{{ $json.package_name }}`
4. Response Format: `JSON` or `String` (depending on what pypistats returns)

**Option B: Code Node (Web Scraping)**

```javascript
const packageName = $input.item.json.package_name;
const date = $input.item.json.date; // e.g., "2025-01"

// Fetch the page
const response = await fetch(`https://pypistats.org/packages/${packageName}`);
const html = await response.text();

// Parse HTML to extract statistics
// This will depend on the actual structure of pypistats.org

// Example parsing (adjust based on actual HTML):
const downloadsMatch = html.match(/Downloads.*?(\d{1,3}(?:,\d{3})*)/);
const uniqueMatch = html.match(/Unique.*?(\d{1,3}(?:,\d{3})*)/);

const downloads = downloadsMatch ? parseInt(downloadsMatch[1].replace(/,/g, '')) : null;
const unique_downloads = uniqueMatch ? parseInt(uniqueMatch[1].replace(/,/g, '')) : null;

return {
  json: {
    downloads,
    unique_downloads,
    package_name: packageName,
    date
  }
};
```

### Step 4: Add Respond to Workflow Node

1. Add **Respond to Workflow** node
2. Connect it to the Code/HTTP node
3. This sends the output back to the calling workflow

### Step 5: Save & Activate

1. Save the workflow
2. Toggle **Active** to enable it
3. Note the **Workflow ID** (needed for calling from other workflows)

## Testing the Sub-Workflow

Create a separate test workflow to verify it works:

### Test Workflow Structure

1. **Manual Trigger** node
   - Manually run the test

2. **Set** node (optional)
   - Set test data:
     ```json
     {
       "package_name": "couchbase",
       "date": "2025-01"
     }
     ```

3. **Execute Workflow** node
   - Workflow: Select "PyPI Stats Sub-Workflow"
   - Input: Pass `package_name` and `date`

4. **Display Result** (use a Code node or just view execution output)

### Testing Steps

1. Create test workflow in n8n
2. Configure Execute Workflow node to call the sub-workflow
3. Run with test data:
   - Package: `couchbase`
   - Date: `2025-01` (or current month)
4. Verify output contains:
   - `downloads` (integer)
   - `unique_downloads` (integer)
5. Try another package:
   - Package: `langchain-couchbase`
   - Date: `2025-01`

### Expected Output

```json
{
  "downloads": 45123,
  "unique_downloads": 38456,
  "package_name": "couchbase",
  "date": "2025-01"
}
```

## Data Limitations

### 3-Month Window

PyPI Stats typically shows data for approximately the **last 3 months**. Requesting statistics for older dates may:
- Return no data
- Return null values
- Cause the workflow to fail

**Recommendation**: Always request recent months only (current month or previous 1-2 months)

### Data Availability

Statistics update with a delay:
- **Current month**: Partial data (month in progress)
- **Previous month**: Complete data (best to query on 1st of next month)
- **Older months**: May be archived or unavailable

## Troubleshooting

### Issue: No data returned

**Possible causes**:
- Package name spelled incorrectly
- Package doesn't exist on PyPI
- Date too old (beyond 3-month window)
- pypistats.org changed their HTML structure

**Solutions**:
- Manually visit `https://pypistats.org/packages/{package_name}` to verify
- Check package name spelling on PyPI.org
- Use a more recent date

### Issue: Parsing errors

**Cause**: HTML structure changed on pypistats.org

**Solution**:
1. Visit the pypistats page manually
2. Inspect HTML structure
3. Update selectors/regex in Code node

### Issue: Rate limiting

**Cause**: Too many requests to pypistats.org

**Solution**:
- Add delays between requests in calling workflow
- Cache results to avoid repeated fetches
- Check pypistats.org robots.txt for rate limits

## Alternative: PyPI Stats API

PyPI Stats may provide an API. Check these resources:

- **pypistats.org API documentation** (if available)
- **pypistats Python library**: <https://github.com/hugovk/pypistats>
- **PyPI JSON API**: `https://pypi.org/pypi/{package}/json` (has different data, not download stats)

If an API exists, prefer it over web scraping for:
- Reliability
- Performance
- Maintainability

## Integration with Task 4

This sub-workflow is called by **Task 4: Package Stats Aggregator** which:
1. Reads list of packages from n8n Data Table
2. Loops through each package
3. Calls this sub-workflow for each package
4. Aggregates results and writes to Google Sheets

**Example call from Task 4**:

```
Execute Workflow Node:
  - Workflow: PyPI Stats Sub-Workflow
  - package_name: {{ $json.package_name }}
  - date: {{ $today.format('YYYY-MM') }}
```

## Maintenance

### Monthly Checks

- [ ] Verify sub-workflow is active
- [ ] Test with current package list (`couchbase`, `langchain-couchbase`)
- [ ] Check pypistats.org is accessible
- [ ] Verify data format hasn't changed

### When pypistats.org Updates

If the website structure changes:
1. Manually inspect the new HTML
2. Update parsing logic in Code node
3. Re-test with all packages
4. Update this README with new selectors

## Future Improvements

1. **Error handling**: Return graceful errors if package not found
2. **Caching**: Cache results to avoid re-fetching same data
3. **Historical data**: Store results in n8n database for long-term tracking
4. **Validation**: Check if `date` parameter is within 3-month window before fetching

## Resources

- [PyPI Stats](https://pypistats.org/)
- [pypistats Python library](https://github.com/hugovk/pypistats)
- [n8n Sub-Workflows Documentation](https://docs.n8n.io/workflows/sub-workflows/)
- [n8n Execute Workflow Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow/)

## Export Workflow

After configuring the sub-workflow:

1. Open the workflow in n8n
2. Click **⋮** (three dots) → **Download**
3. Save as `pypistats-subworkflow.json` in this directory
4. Commit to git

Also export the test workflow as `pypistats-test-workflow.json`
