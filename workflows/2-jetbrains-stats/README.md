# Task 2: JetBrains Plugin Statistics Workflow (DA-1263)

## Overview

This workflow automatically extracts download statistics for the **Couchbase IntelliJ Plugin** from JetBrains Marketplace and stores them in Google Sheets for historical tracking.

**Target Plugin**: [Couchbase - IntelliJ IDEs Plugin](https://plugins.jetbrains.com/plugin/22131-couchbase)

**Metrics Collected**:
- Total downloads (all-time)
- Downloads this month (current/previous month depending on run time)

**Schedule**: Runs on the 1st of each month to capture previous month's statistics

## Workflow Structure

### Triggers

1. **Manual Trigger** - For testing and ad-hoc runs
2. **Schedule Trigger** - 1st of the month, 9:00 AM UTC

### Nodes

1. **Trigger Node** - Manual or scheduled
2. **Code Node (Puppeteer)** - Browser automation to scrape JetBrains
3. **Google Sheets Node** - Write statistics to spreadsheet

### Data Storage

**Google Sheets**: <https://docs.google.com/spreadsheets/d/1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8/>

**Tab**: `jetbrains`

**Columns**:
| Year-Month | Total Downloads | Downloads This Month | Date Collected |
|------------|-----------------|----------------------|----------------|
| 2025-01    | 15234          | 342                  | 2025-02-01     |

## Manual Flow (For Testing)

Before automating with Puppeteer, you can manually verify the flow:

1. Go to <https://account.jetbrains.com/login>
2. Click "Continue with email"
3. Enter your JetBrains email and password
4. Once authenticated, navigate to: <https://plugins.jetbrains.com/plugin/22131-couchbase/edit/analytics>
5. Accept cookie banner if prompted
6. If asked to login again, click "Login with JetBrains Account"
7. View the statistics on the analytics page

## Puppeteer Implementation

### Standalone Script (Node.js)

This script works outside of n8n for testing purposes:

```javascript
const puppeteer = require('puppeteer');

const EMAIL = 'your-jetbrains-email@example.com';
const PASSWORD = 'your-password';

const sleep = (ms) => new Promise(r => setTimeout(r, ms));

(async () => {
  let browser;
  try {
    const browser = await puppeteer.launch({
      headless: false,
      defaultViewport: null,
      args: ['--start-maximized'],
      slowMo: 50,
    });

    const page = await browser.newPage();
    page.setDefaultTimeout(60000);
    page.setDefaultNavigationTimeout(60000);

    const log = (...a) => console.log(new Date().toISOString(), ...a);

    // Helper functions
    async function waitVisible(page, sel, timeout = 30000) {
      await page.waitForSelector(sel, { visible: true, timeout });
      await page.waitForFunction((s) => {
        const el = document.querySelector(s);
        if (!el) return false;
        const r = el.getBoundingClientRect();
        const st = getComputedStyle(el);
        return r.width > 0 && r.height > 0 && st.visibility !== 'hidden' && st.display !== 'none';
      }, {}, sel);
    }

    async function typeInto(page, sel, value) {
      await waitVisible(page, sel);
      const el = await page.$(sel);
      await el.click({ clickCount: 3 });
      await el.type(value, { delay: 12 });
    }

    async function submitFormForInput(page, inputSelector) {
      await page.$eval(inputSelector, (input) => {
        const form = input.closest('form');
        if (!form) throw new Error('Form not found for input ' + inputSelector);
        const btn = form.querySelector('button[type="submit"], button[data-test="button"], [type="submit"]');
        if (btn && !btn.disabled && btn.getAttribute('aria-disabled') !== 'true') {
          btn.scrollIntoView({ block: 'center', inline: 'center' });
          btn.click();
        } else {
          form.requestSubmit ? form.requestSubmit() : form.submit();
        }
      });
    }

    // PHASE 1: JetBrains Account login
    log('goto /login');
    await page.goto('https://account.jetbrains.com/login', { waitUntil: 'domcontentloaded' });

    const emailPresent = await page.evaluate(() => !!document.querySelector('input[name="email"]'));
    if (!emailPresent) {
      await page.evaluate(() => {
        const norm = (s) => (s || '').replace(/\s+/g, ' ').trim().toLowerCase();
        const btn = [...document.querySelectorAll('button')].find(b => {
          const t = norm(b.textContent);
          return t.includes('continue') && t.includes('email');
        });
        if (btn) { btn.scrollIntoView({ block: 'center' }); btn.click(); }
      });
      await sleep(1000);
    }

    log('email');
    await typeInto(page, 'input[name="email"]', EMAIL);
    await submitFormForInput(page, 'input[name="email"]');
    await sleep(2000);

    log('password');
    await typeInto(page, 'input[name="password"]', PASSWORD);
    await submitFormForInput(page, 'input[name="password"]');
    await sleep(3000);

    // PHASE 2: Navigate to analytics page
    log('goto analytics editor');
    await page.goto('https://plugins.jetbrains.com/plugin/22131-couchbase/edit/analytics', { waitUntil: 'networkidle2' });

    // Accept cookies
    await page.evaluate(() => {
      const btn = document.querySelector('button.ch2-btn.ch2-allow-all-btn.ch2-btn-primary');
      if (btn) btn.click();
    }).catch(() => {});
    await sleep(2000);

    // PHASE 3: Extract metrics
    log('extract metrics');

    await page.waitForFunction(() => {
      const txt = document.body.innerText || '';
      return txt.includes('Total downloads') && txt.includes('Downloads');
    }, { timeout: 30000 }).catch(() => {});

    const { totalDownloads, downloadsThisMonth } = await page.evaluate(() => {
      const normNum = (txt) => {
        const digits = (txt || '').replace(/[^\d]/g, '');
        return digits ? parseInt(digits, 10) : null;
      };

      let total = null;
      let month = null;

      const divs = [...document.querySelectorAll('div')];

      for (const d of divs) {
        const txt = (d.textContent || '').replace(/\s+/g, ' ').trim();

        if (/^Total downloads/i.test(txt)) {
          const h3 = d.querySelector('h3');
          if (h3) total = normNum(h3.textContent);
        }

        if (/^Downloads this month/i.test(txt)) {
          const h3 = d.querySelector('h3');
          if (h3) month = normNum(h3.textContent);
        }
      }

      return { totalDownloads: total, downloadsThisMonth: month };
    });

    log('DONE', {
      url: page.url(),
      totalDownloads,
      downloadsThisMonth
    });

    await browser.close();
  } catch (err) {
    console.error('FATAL ERROR:', err);
    if (browser) await browser.close().catch(() => {});
  }
})();
```

### n8n Code Node Script

For use in n8n workflows (uses `$page` instead of creating browser instance):

**Note**: See `tasks/2.txt` for the complete n8n script with advanced cookie handling and Hub authentication logic.

Key differences from standalone:
- No `require('puppeteer')` - n8n provides `$page`
- Returns array with results: `return [{ totalDownloads, downloadsThisMonth }]`
- Includes screenshot capture for debugging: `const screenshot_base64 = await $page.screenshot({ fullPage: true })`

## Known Issues

### Issue 1: Cookie Banner Not Dismissing

**Status**: Currently failing

**Symptoms**: The workflow fails to accept the CookieHub (ch2) cookie banner, preventing access to statistics

**Debugging**:
- The workflow captures a base64 screenshot for debugging
- Decode the `screenshot_base64` field using an online tool (<https://base64.guru/converter/decode/image>)
- Visual inspection shows whether cookie banner is still visible

**Attempted Solutions** (in n8n script):
1. Clicking "Accept All" button with mouse/pointer events
2. Keyboard "Enter" event on focused button
3. Calling CookieHub API methods (`setConsentWithAll`, `acceptAll`, `closeDialog`)
4. Clicking dismiss/close button
5. Force-hiding dialog via DOM manipulation

**Potential Fixes**:
- Wait longer after page load before attempting cookie dismissal
- Try clicking via Puppeteer's native `page.click()` instead of `evaluate()`
- Check if JetBrains updated their cookie consent implementation
- Use `page.setRequestInterception()` to block CookieHub scripts entirely

### Issue 2: Hub Authentication Redirect Loop

**Symptoms**: After logging into JetBrains Account, redirect to Hub authentication doesn't complete

**Mitigation**: Script navigates via `window.location.assign(href)` instead of clicking buttons (more reliable)

### Issue 3: Metrics Not Found

**Symptoms**: `totalDownloads` or `downloadsThisMonth` returns `null`

**Possible Causes**:
- Page not fully loaded before extraction
- JetBrains changed the HTML structure
- Not properly authenticated

**Debug Steps**:
1. Check screenshot_base64 to see actual page state
2. Manually navigate to analytics page to verify structure
3. Inspect HTML for new selectors/class names

## Required Credentials

### JetBrains Account

Store in n8n **Settings** → **Credentials**:

**Type**: Custom credential (or HTTP Basic Auth)

**Fields**:
- Email: `your-jetbrains-account@example.com`
- Password: `your-password`

**Note**: Replace `EMAIL` and `PASSWORD` constants in the Code node with credential references

### Google Sheets

**Type**: Google Sheets OAuth2 or Service Account

**Required Scopes**:
- `https://www.googleapis.com/auth/spreadsheets`

**Setup**:
1. Create a Google Cloud project
2. Enable Google Sheets API
3. Create OAuth2 credentials or Service Account
4. Share the spreadsheet with the service account email (if using service account)

## Workflow Configuration

### Step 1: Create Workflow in n8n

1. Open n8n: <https://kaustavcouchbase.app.n8n.cloud>
2. Click **Workflows** → **Add Workflow**
3. Name it: "JetBrains Plugin Stats"

### Step 2: Add Trigger Nodes

**Manual Trigger**:
- Add **Manual Trigger** node

**Schedule Trigger**:
- Add **Schedule Trigger** node
- Cron: `0 9 1 * *` (1st of month, 9:00 AM UTC)

### Step 3: Add Code Node (Puppeteer)

1. Add **Code** node
2. Set Mode to "Run Once for All Items"
3. Paste the n8n Puppeteer script from `tasks/2.txt`
4. Replace `EMAIL` and `PASSWORD` with credential references:
   ```javascript
   const EMAIL = this.getCredentials('jetbrainsAccount').email;
   const PASSWORD = this.getCredentials('jetbrainsAccount').password;
   ```

### Step 4: Add Google Sheets Node

1. Add **Google Sheets** node
2. Set Operation to "Append" or "Update"
3. Set Spreadsheet ID: `1s7DTSNKtBbXTUQRnjSr5nNh2ihnLVFJUEpBSiR90-i8`
4. Set Sheet Name: `jetbrains`
5. Map fields:
   - Column A: Year-Month (calculate from `new Date()`)
   - Column B: `{{ $json.totalDownloads }}`
   - Column C: `{{ $json.downloadsThisMonth }}`
   - Column D: `{{ new Date().toISOString() }}`

### Step 5: Test

1. Click **Execute Workflow** (manual trigger)
2. Monitor execution
3. Check screenshot_base64 if errors occur
4. Verify data written to Google Sheets

### Step 6: Activate Schedule

Once testing succeeds:
1. Click **Active** toggle to enable the workflow
2. The schedule trigger will run automatically on the 1st of each month

## Troubleshooting

### Workflow times out

**Solution**: Increase timeouts in Code node:
```javascript
$page.setDefaultTimeout(120000); // 2 minutes
$page.setDefaultNavigationTimeout(120000);
```

### Screenshots show wrong page

**Solution**: Add more wait time after navigation:
```javascript
await sleep(3000); // Wait 3 seconds for page to load
```

### Credentials error

**Solution**: Verify credentials are properly configured and referenced in Code node

### Google Sheets permission denied

**Solution**: Share spreadsheet with service account email or verify OAuth2 scopes

## Maintenance

### When JetBrains Updates Their UI

If the workflow breaks due to UI changes:

1. Manually navigate to the analytics page
2. Inspect the HTML structure
3. Update selectors in the Code node:
   - Cookie banner button: `button.ch2-btn.ch2-allow-all-btn`
   - Total downloads label: Text matching `/^Total downloads/i`
   - Downloads this month label: Text matching `/^Downloads this month/i`
   - Metrics values: `h3` elements near labels

### Monthly Review

Check these on the 1st of each month after automated run:

- [ ] Workflow executed successfully
- [ ] Data written to Google Sheets
- [ ] Values look reasonable (not null, increasing total downloads)
- [ ] No error notifications from n8n

## Future Improvements

1. **Add retry logic**: If metrics extraction fails, retry 2-3 times
2. **Email notifications**: Send alert if workflow fails
3. **Historical comparison**: Compare month-over-month growth
4. **Additional metrics**: Extract download breakdown by IDE version
5. **Remove cookie banner dependency**: Use API if JetBrains provides one

## Resources

- [JetBrains Marketplace](https://plugins.jetbrains.com/)
- [Couchbase IntelliJ Plugin](https://plugins.jetbrains.com/plugin/22131-couchbase)
- [Puppeteer Documentation](https://pptr.dev/)
- [n8n Code Node Docs](https://docs.n8n.io/code-examples/methods-variables-examples/)
- Full script in: `tasks/2.txt`

## Export Workflow

After configuring the workflow in n8n:

1. Open the workflow
2. Click **⋮** (three dots) → **Download**
3. Save as `jetbrains-stats.json` in this directory
4. Commit to git
