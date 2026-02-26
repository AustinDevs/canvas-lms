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

### 2. Deploy This Tool

```bash
cd tools/metabase-lti
cp .env.example .env
# Edit .env with your actual values
npm install
npm start
```

The tool needs to be accessible from both Canvas server and user browsers.
For production, deploy behind a reverse proxy with HTTPS.

### 3. Add to Canvas

#### Option A: Auto-configure via XML (recommended)

1. Go to **Canvas Admin > Settings > Apps > + App**
2. Configuration Type: **By URL**
3. Consumer Key: (same as `LTI_CONSUMER_KEY` in your `.env`)
4. Shared Secret: (same as `LTI_CONSUMER_SECRET` in your `.env`)
5. Config URL: `https://your-tool-host/config.xml`
6. Submit

#### Option B: Manual configuration

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

### 4. Add Metabase Domain to Canvas CSP (if CSP is enabled)

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

Add the tool to course navigation with custom parameters:

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
