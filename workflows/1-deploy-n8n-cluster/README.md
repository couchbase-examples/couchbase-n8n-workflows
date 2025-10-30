# Task 1: Deploy N8N Cluster (DA-1261)

## Overview

This task covers the initial deployment and configuration of the n8n cluster for the Couchbase DevEx team. The cluster is already deployed and accessible at:

**n8n Cloud Instance**: <https://kaustavcouchbase.app.n8n.cloud>

## Requirements

### Docker Image with Puppeteer Support

Since many of our workflows require browser automation (scraping JetBrains Marketplace, navigating authenticated pages, etc.), we need n8n with Puppeteer capabilities.

**Recommended Image**: `n8nio/n8n:latest` with Puppeteer nodes installed

**Alternative**: [drudge/n8n-nodes-puppeteer](https://github.com/drudge/n8n-nodes-puppeteer) - Community package for Puppeteer integration

### Installing Puppeteer Nodes in n8n Cloud

Since we're using n8n Cloud (not self-hosted), Puppeteer support may need to be enabled:

1. Contact n8n support to enable custom community nodes
2. Alternatively, use the built-in **Code node** with Puppeteer scripts (see Task 2 for examples)
3. For self-hosted deployments, use Docker with Puppeteer pre-installed

## User Management

### Team Members to Add

Create n8n user accounts for all members of the DevEx team:

- **Niranjan** - DevEx team member
- **Vishal** - DevEx team member
- **Priya** - DevEx team member
- **Kaustav** - Already has access (instance owner)
- **Dima** - DevEx team lead

### Adding Users in n8n Cloud

1. Log into n8n Cloud as admin: <https://kaustavcouchbase.app.n8n.cloud>
2. Navigate to **Settings** → **Users**
3. Click **Invite User**
4. Enter user email address
5. Select role:
   - **Owner**: Full access (admin)
   - **Member**: Can create/edit workflows
   - **Viewer**: Read-only access
6. Send invitation
7. User receives email and sets up their account

**Recommended Role**: **Member** for all DevEx team members

## Deployment Architecture

### n8n Cloud (Current Setup)

- **Hosting**: Managed by n8n.io
- **URL**: <https://kaustavcouchbase.app.n8n.cloud>
- **Advantages**:
  - No infrastructure management
  - Automatic updates
  - Built-in monitoring
  - Easy scaling

### Self-Hosted Alternative (Future Option)

If we need more control or custom Docker images with Puppeteer:

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=admin \
  -e N8N_BASIC_AUTH_PASSWORD=your-password \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n:latest
```

**With Puppeteer**:

```dockerfile
FROM n8nio/n8n:latest

USER root

# Install Chromium and dependencies for Puppeteer
RUN apk add --no-cache \
    chromium \
    nss \
    freetype \
    harfbuzz \
    ca-certificates \
    ttf-freefont

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true \
    PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser

USER node
```

## Configuration

### Environment Variables

Key environment variables for n8n configuration:

```bash
# Basic Auth (if self-hosted)
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secure-password

# Webhooks (for external triggers)
N8N_HOST=kaustavcouchbase.app.n8n.cloud
N8N_PROTOCOL=https

# Execution Mode
EXECUTIONS_MODE=regular

# Timezone
GENERIC_TIMEZONE=America/Los_Angeles
```

### Required Credentials

Set up these credentials in n8n for workflows to function:

1. **JetBrains Account** (for Task 2)
   - Type: Custom credential with email/password
   - Used for: Authenticating to JetBrains Marketplace

2. **Google Sheets API** (for Tasks 2, 4)
   - Type: OAuth2 or Service Account
   - Used for: Writing statistics to spreadsheets
   - Required scopes: `spreadsheets`, `spreadsheets.readonly`

3. **n8n API Key** (for Claude Code integration)
   - Type: API Key
   - Used for: Claude Code MCP server to manage workflows

## Verification

### Check Deployment Status

1. Access n8n Cloud instance: <https://kaustavcouchbase.app.n8n.cloud>
2. Verify login works
3. Check **Settings** → **About** to confirm version

### Test Puppeteer Availability

Create a test workflow with a Code node:

```javascript
// Simple Puppeteer test
const puppeteer = require('puppeteer');

const browser = await puppeteer.launch({
  headless: true,
  args: ['--no-sandbox', '--disable-setuid-sandbox']
});

const page = await browser.newPage();
await page.goto('https://example.com');
const title = await page.title();

await browser.close();

return { title };
```

If this executes successfully, Puppeteer is available.

## Troubleshooting

### Issue: Puppeteer not available in n8n Cloud

**Solution**: Use Code nodes with Puppeteer npm package. n8n Cloud has Puppeteer pre-installed in the execution environment.

### Issue: Users can't access instance

**Solution**:
- Verify invitation email was sent
- Check spam folder
- Resend invitation from Settings → Users
- Ensure user clicked activation link

### Issue: Workflows fail with browser errors

**Solution**:
- Check Puppeteer launch args include `--no-sandbox`
- Increase timeout values in Code nodes
- Review screenshot outputs for debugging (see Task 2)

## Next Steps

1. Verify all team members have access
2. Import workflows from this repository (see `/docs/getting-started.md`)
3. Configure required credentials in Settings → Credentials
4. Test each workflow with manual triggers before scheduling

## Resources

- [n8n Documentation](https://docs.n8n.io/)
- [n8n Puppeteer Community Node](https://github.com/drudge/n8n-nodes-puppeteer)
- [Puppeteer Documentation](https://pptr.dev/)
- [n8n Cloud User Management](https://docs.n8n.io/hosting/cloud/user-management/)

## Task Status

- [x] n8n Cloud instance deployed
- [x] Puppeteer support verified
- [ ] All team members added (Niranjan, Vishal, Priya)
- [ ] Required credentials configured
- [ ] Test workflows imported and verified
