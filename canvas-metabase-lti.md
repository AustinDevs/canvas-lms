# Canvas Metabase LTI Tool

A standalone LTI 1.1 external tool that embeds Metabase dashboards and questions
into Canvas LMS using Metabase's guest embedding feature.

**Zero changes to the Canvas codebase required.**

## How It Works

1. Canvas launches this tool via LTI (in an iframe)
2. The tool generates a signed JWT using your Metabase embedding secret
3. It renders a page with Metabase's guest embed web components
4. The Metabase dashboard/question appears inside Canvas

## Prerequisites

- Node.js 18+
- A Metabase instance with guest embedding enabled
- Canvas LMS admin access (to add the external tool)

## Setup

### 1. Configure Metabase

1. Go to **Metabase Admin > Embedding**
2. Enable **Guest embedding**
3. Copy the **Embedding secret key**
4. Enable specific dashboards/questions for embedding

### 2. Create the project

```bash
mkdir canvas-metabase-lti && cd canvas-metabase-lti
```

### 3. Create files

#### `package.json`

```json
{
  "name": "canvas-metabase-lti",
  "version": "1.0.0",
  "description": "LTI tool that embeds Metabase dashboards and questions into Canvas LMS via guest embedding",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js"
  },
  "dependencies": {
    "express": "^4.21.0",
    "jsonwebtoken": "^9.0.2",
    "ims-lti": "^3.0.2",
    "dotenv": "^16.4.5"
  }
}
```

#### `.env.example`

```bash
# --- LTI Configuration ---
# These are used to authenticate launches from Canvas.
# Set the same key/secret when adding the external tool in Canvas.
LTI_CONSUMER_KEY=metabase-lti
LTI_CONSUMER_SECRET=change-me-to-a-secure-secret

# --- Metabase Configuration ---
# Your Metabase instance URL (no trailing slash)
METABASE_SITE_URL=https://metabase.example.com

# Metabase embedding secret key
# Find this in Metabase Admin > Embedding > Guest embedding
METABASE_EMBEDDING_SECRET=your-metabase-embedding-secret-here

# --- Server ---
PORT=3001
```

#### `.gitignore`

```
node_modules/
.env
```

#### `server.js`

```javascript
require('dotenv').config();
const express = require('express');
const lti = require('ims-lti');
const jwt = require('jsonwebtoken');

const app = express();
app.use(express.urlencoded({ extended: true }));

const {
  PORT = 3001,
  LTI_CONSUMER_KEY,
  LTI_CONSUMER_SECRET,
  METABASE_SITE_URL,
  METABASE_EMBEDDING_SECRET,
} = process.env;

// Validate required config at startup
const required = { LTI_CONSUMER_KEY, LTI_CONSUMER_SECRET, METABASE_SITE_URL, METABASE_EMBEDDING_SECRET };
for (const [key, val] of Object.entries(required)) {
  if (!val) {
    console.error(`Missing required environment variable: ${key}`);
    process.exit(1);
  }
}

// Health check
app.get('/health', (_req, res) => res.json({ status: 'ok' }));

// LTI tool configuration XML — Canvas uses this for auto-configuration
app.get('/config.xml', (req, res) => {
  const host = `${req.protocol}://${req.get('host')}`;
  res.type('application/xml').send(`<?xml version="1.0" encoding="UTF-8"?>
<cartridge_basiclti_link
  xmlns="http://www.imsglobal.org/xsd/imslticc_v1p0"
  xmlns:blti="http://www.imsglobal.org/xsd/imsbasiclti_v1p0"
  xmlns:lticm="http://www.imsglobal.org/xsd/imslticm_v1p0"
  xmlns:lticp="http://www.imsglobal.org/xsd/imslticp_v1p0"
  xmlns:lti="http://www.imsglobal.org/xsd/imsltiext_v1p0">
  <blti:title>Metabase Dashboard</blti:title>
  <blti:description>Embed Metabase dashboards and questions via guest embedding</blti:description>
  <blti:launch_url>${host}/launch</blti:launch_url>
  <blti:extensions platform="canvas.instructure.com">
    <lticm:property name="privacy_level">anonymous</lticm:property>
    <lticm:property name="domain">${req.get('host')}</lticm:property>
    <lticm:options name="course_navigation">
      <lticm:property name="enabled">false</lticm:property>
      <lticm:property name="text">Metabase Dashboard</lticm:property>
    </lticm:options>
    <lticm:options name="editor_button">
      <lticm:property name="enabled">true</lticm:property>
      <lticm:property name="text">Embed Metabase</lticm:property>
      <lticm:property name="icon_url">${host}/icon.svg</lticm:property>
      <lticm:property name="message_type">ContentItemSelectionRequest</lticm:property>
      <lticm:property name="url">${host}/select</lticm:property>
    </lticm:options>
  </blti:extensions>
  <cartridge_bundle identifierref="BLTI001_Bundle"/>
  <cartridge_icon identifierref="BLTI001_Icon"/>
</cartridge_basiclti_link>`);
});

// Simple icon for the editor button
app.get('/icon.svg', (_req, res) => {
  res.type('image/svg+xml').send(`<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" width="24" height="24">
  <rect width="24" height="24" rx="4" fill="#509EE3"/>
  <text x="12" y="17" text-anchor="middle" fill="white" font-size="14" font-family="sans-serif" font-weight="bold">M</text>
</svg>`);
});

/**
 * Validate an LTI 1.1 launch request.
 * Returns a promise that resolves to the LTI provider if valid.
 */
function validateLtiLaunch(req) {
  return new Promise((resolve, reject) => {
    const provider = new lti.Provider(LTI_CONSUMER_KEY, LTI_CONSUMER_SECRET);
    provider.valid_request(req, req.body, (err, isValid) => {
      if (err || !isValid) {
        reject(err || new Error('Invalid LTI launch'));
      } else {
        resolve(provider);
      }
    });
  });
}

/**
 * Generate a Metabase guest embedding JWT for a dashboard or question.
 */
function generateMetabaseToken(resourceType, resourceId, params = {}) {
  const payload = {
    resource: { [resourceType]: resourceId },
    params,
    exp: Math.floor(Date.now() / 1000) + 600, // 10 min expiry
  };
  return jwt.sign(payload, METABASE_EMBEDDING_SECRET);
}

/**
 * Build the HTML page that renders the Metabase guest embed.
 */
function buildEmbedPage(resourceType, resourceId, params = {}, title = 'Metabase') {
  const token = generateMetabaseToken(resourceType, resourceId, params);
  const tagName = resourceType === 'dashboard' ? 'metabase-dashboard' : 'metabase-question';

  return `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>${title}</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { width: 100%; height: 100%; overflow: hidden; }
    ${tagName} { display: block; width: 100%; height: 100vh; }
  </style>
</head>
<body>
  <${tagName} id="metabase-embed"></${tagName}>
  <script src="${METABASE_SITE_URL}/app/embed.js"><\/script>
  <script>
    window.metabaseConfig = {
      metabaseInstanceUrl: "${METABASE_SITE_URL}",
      guest: true
    };
    document.getElementById("metabase-embed").setAttribute("jwt", ${JSON.stringify(token)});
  <\/script>
</body>
</html>`;
}

/**
 * Direct launch — embed a specific dashboard or question.
 *
 * Custom parameters (set in Canvas tool config):
 *   custom_metabase_type      = "dashboard" or "question" (default: "dashboard")
 *   custom_metabase_id        = numeric ID of the dashboard/question
 *   custom_metabase_params    = JSON-encoded locked parameters (optional)
 *   custom_metabase_title     = display title (optional)
 *
 * Or pass via query string for testing: ?type=dashboard&id=1
 */
app.post('/launch', async (req, res) => {
  try {
    await validateLtiLaunch(req);
  } catch (err) {
    console.error('LTI validation failed:', err.message);
    return res.status(401).send('Unauthorized: Invalid LTI launch');
  }

  const resourceType = req.body.custom_metabase_type || 'dashboard';
  const resourceId = parseInt(req.body.custom_metabase_id, 10);
  const title = req.body.custom_metabase_title || 'Metabase';
  let params = {};

  if (req.body.custom_metabase_params) {
    try {
      params = JSON.parse(req.body.custom_metabase_params);
    } catch {
      // ignore malformed params
    }
  }

  if (!resourceId) {
    return res.status(400).send(`
      <h2>Configuration Error</h2>
      <p>No Metabase resource ID provided. Set <code>custom_metabase_id</code>
      in the LTI tool custom parameters.</p>
      <p>Example: <code>metabase_id=1</code></p>
    `);
  }

  res.send(buildEmbedPage(resourceType, resourceId, params, title));
});

/**
 * Content Item Selection — lets instructors pick a dashboard/question
 * to embed directly into a Canvas page via the RCE editor button.
 */
app.post('/select', async (req, res) => {
  try {
    await validateLtiLaunch(req);
  } catch (err) {
    console.error('LTI validation failed:', err.message);
    return res.status(401).send('Unauthorized: Invalid LTI launch');
  }

  const returnUrl = req.body.content_item_return_url;
  const host = `${req.protocol}://${req.get('host')}`;

  res.send(`<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Select Metabase Resource</title>
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; max-width: 500px; margin: 40px auto; padding: 0 20px; }
    h2 { margin-bottom: 20px; }
    label { display: block; margin: 12px 0 4px; font-weight: 600; }
    input, select { width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; font-size: 14px; }
    button { margin-top: 20px; padding: 10px 24px; background: #509EE3; color: #fff; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
    button:hover { background: #3d7ec2; }
    .help { font-size: 12px; color: #666; margin-top: 2px; }
  </style>
</head>
<body>
  <h2>Embed Metabase Resource</h2>
  <form id="selectForm">
    <label for="type">Resource Type</label>
    <select id="type" name="type">
      <option value="dashboard">Dashboard</option>
      <option value="question">Question</option>
    </select>

    <label for="resourceId">Resource ID</label>
    <input type="number" id="resourceId" name="resourceId" min="1" required placeholder="e.g. 1">
    <div class="help">Find this in the Metabase URL: /dashboard/<b>1</b> or /question/<b>42</b></div>

    <label for="title">Title (optional)</label>
    <input type="text" id="title" name="title" placeholder="My Dashboard">

    <label for="width">Width</label>
    <input type="text" id="width" name="width" value="100%" placeholder="100%">

    <label for="height">Height</label>
    <input type="text" id="height" name="height" value="600px" placeholder="600px">

    <button type="submit">Embed</button>
  </form>

  <script>
    document.getElementById('selectForm').addEventListener('submit', function(e) {
      e.preventDefault();
      var type = document.getElementById('type').value;
      var id = document.getElementById('resourceId').value;
      var title = document.getElementById('title').value || 'Metabase ' + type + ' ' + id;
      var width = document.getElementById('width').value;
      var height = document.getElementById('height').value;

      var launchUrl = '${host}/launch';
      var iframeHtml = '<iframe src="' + launchUrl + '?type=' + type + '&id=' + id + '"'
        + ' width="' + width + '" height="' + height + '"'
        + ' style="border:0;" title="' + title + '"'
        + ' allowfullscreen loading="lazy"></iframe>';

      // Build Content Item response
      var form = document.createElement('form');
      form.method = 'POST';
      form.action = ${JSON.stringify(returnUrl)};

      var items = {
        "@context": "http://purl.imsglobal.org/ctx/lti/v1/ContentItem",
        "@graph": [{
          "@type": "ContentItem",
          "mediaType": "text/html",
          "text": iframeHtml,
          "placementAdvice": { "presentationDocumentTarget": "embed" },
          "title": title
        }]
      };

      var fields = {
        "lti_message_type": "ContentItemSelection",
        "lti_version": "LTI-1p0",
        "content_items": JSON.stringify(items)
      };

      Object.keys(fields).forEach(function(key) {
        var input = document.createElement('input');
        input.type = 'hidden';
        input.name = key;
        input.value = fields[key];
        form.appendChild(input);
      });

      document.body.appendChild(form);
      form.submit();
    });
  <\/script>
</body>
</html>`);
});

/**
 * Direct embed endpoint — used by iframes inserted via Content Items.
 * Generates a fresh JWT each time the page loads.
 */
app.get('/launch', (req, res) => {
  const resourceType = req.query.type || 'dashboard';
  const resourceId = parseInt(req.query.id, 10);

  if (!resourceId) {
    return res.status(400).send('Missing resource id');
  }

  let params = {};
  if (req.query.params) {
    try {
      params = JSON.parse(req.query.params);
    } catch {
      // ignore
    }
  }

  res.send(buildEmbedPage(resourceType, resourceId, params, `Metabase ${resourceType} ${resourceId}`));
});

app.listen(PORT, () => {
  console.log(`Metabase LTI tool listening on port ${PORT}`);
  console.log(`Config XML: http://localhost:${PORT}/config.xml`);
  console.log(`Health:     http://localhost:${PORT}/health`);
});
```

## Deploy and run

```bash
cp .env.example .env
# Edit .env with your actual values
npm install
npm start
```

The tool needs to be accessible from both Canvas server and user browsers.
For production, deploy behind a reverse proxy with HTTPS.

## Add to Canvas

### Option A: Auto-configure via XML (recommended)

1. Go to **Canvas Admin > Settings > Apps > + App**
2. Configuration Type: **By URL**
3. Consumer Key: (same as `LTI_CONSUMER_KEY` in your `.env`)
4. Shared Secret: (same as `LTI_CONSUMER_SECRET` in your `.env`)
5. Config URL: `https://your-tool-host/config.xml`
6. Submit

### Option B: Manual configuration

1. Go to **Canvas Admin > Settings > Apps > + App**
2. Configuration Type: **Manual Entry**
3. Name: `Metabase Dashboard`
4. Consumer Key / Shared Secret: (from your `.env`)
5. Launch URL: `https://your-tool-host/launch`
6. Set custom fields:
   ```
   metabase_type=dashboard
   metabase_id=YOUR_DASHBOARD_ID
   ```

### Add Metabase Domain to Canvas CSP (if CSP is enabled)

If your Canvas instance has Content Security Policy enabled:

1. Go to **Canvas Admin > Security**
2. Add your Metabase domain (e.g. `metabase.example.com`) to the allowed list
3. Also add the tool's domain if different

## Usage

### Embedding via the RCE Editor Button

If you used the XML config, an "Embed Metabase" button appears in the
Rich Content Editor toolbar:

1. Edit a Canvas page/assignment/etc.
2. Click the **Embed Metabase** button in the editor toolbar
3. Select dashboard or question, enter the ID
4. The embed is inserted into your page

### Embedding via Course Navigation

1. Go to **Course Settings > Navigation**
2. Enable "Metabase Dashboard"
3. In the tool settings, set custom parameters:
   ```
   metabase_type=dashboard
   metabase_id=1
   metabase_title=Course Analytics
   ```

### Embedding via Module Items

1. In a module, click **+** > **External Tool**
2. Select "Metabase Dashboard"
3. Set the URL to: `https://your-tool-host/launch?type=dashboard&id=1`

## Custom Parameters Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `metabase_type` | `dashboard` or `question` | `dashboard` |
| `metabase_id` | Numeric ID from Metabase URL | (required) |
| `metabase_title` | Display title | `Metabase` |
| `metabase_params` | JSON-encoded locked parameters | `{}` |

## Architecture

```
Canvas Page (iframe) ──> This LTI Tool ──> Metabase Guest Embed
                         (JWT signing)     (web components)
```

- Canvas launches the tool in an iframe via LTI
- The tool signs a JWT with the Metabase embedding secret
- The page loads Metabase's `embed.js` and renders the web component
- Users see the dashboard/question inline in Canvas
