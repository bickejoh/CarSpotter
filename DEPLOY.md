# SPOTTER — Netlify Deployment Guide
## Moving from GitHub Pages → Netlify (with working AI)

---

## What you're adding to your repo

```
your-repo/
├── netlify.toml                    ← NEW: tells Netlify how to build
├── netlify/
│   └── functions/
│       └── claude.js               ← NEW: secure API proxy
└── public/
    └── index.html                  ← UPDATED: API calls now go through proxy
```

---

## Step 1 — Get an Anthropic API Key

If you don't have one already:

1. Go to **https://console.anthropic.com**
2. Sign in or create a free account
3. Click **API Keys** in the left sidebar
4. Click **Create Key**, give it a name like "spotter-app"
5. **Copy the key** — it starts with `sk-ant-...`
6. Save it somewhere safe (you won't see it again)

> The free tier includes enough credits to identify hundreds of cars.

---

## Step 2 — Add the files to your GitHub repo

You have three files to add. Do this directly on GitHub.com (no terminal needed):

### Add `netlify.toml` to your repo root

1. Go to your repo on GitHub
2. Click **Add file → Create new file**
3. Name it exactly: `netlify.toml`
4. Paste this content:

```toml
[build]
  publish = "public"
  functions = "netlify/functions"

[[redirects]]
  from = "/api/claude"
  to   = "/.netlify/functions/claude"
  status = 200
```

5. Click **Commit changes**

---

### Add `netlify/functions/claude.js`

1. Click **Add file → Create new file** again
2. In the filename box, type: `netlify/functions/claude.js`
   *(GitHub will automatically create the folders)*
3. Paste this content:

```javascript
exports.handler = async (event) => {
  if (event.httpMethod !== 'POST') {
    return { statusCode: 405, body: JSON.stringify({ error: 'Method not allowed' }) };
  }
  if (!process.env.ANTHROPIC_API_KEY) {
    return { statusCode: 500, body: JSON.stringify({ error: 'API key not configured' }) };
  }
  try {
    const body = JSON.parse(event.body);
    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type':        'application/json',
        'x-api-key':           process.env.ANTHROPIC_API_KEY,
        'anthropic-version':   '2023-06-01',
      },
      body: JSON.stringify(body),
    });
    const data = await response.json();
    return {
      statusCode: response.status,
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
      body: JSON.stringify(data),
    };
  } catch (err) {
    return {
      statusCode: 500,
      headers: { 'Access-Control-Allow-Origin': '*' },
      body: JSON.stringify({ error: 'Proxy error', detail: err.message }),
    };
  }
};
```

4. Click **Commit changes**

---

### Replace your `index.html`

1. Find your current HTML file in the repo (likely `index.html` or inside a `public/` folder)
2. **Replace it entirely** with the new `index.html` from this package
   - Click the file → click the pencil ✏️ edit icon → select all → paste the new content
   - Or delete the old file and upload the new one
3. Click **Commit changes**

---

## Step 3 — Connect to Netlify

1. Go to **https://netlify.com** and sign up for free (use "Sign up with GitHub")
2. Click **Add new site → Import an existing project**
3. Click **Deploy with GitHub**
4. Authorize Netlify to access your GitHub account
5. Find and select your SPOTTER repo
6. On the build settings page:
   - **Branch to deploy:** `main` (or `master`)
   - **Build command:** *(leave blank)*
   - **Publish directory:** `public`
7. Click **Deploy site**

Netlify will give your site a random URL like `magical-cupcake-abc123.netlify.app`. You can change this to something like `spotter-app.netlify.app` in Site Settings → General → Site name.

---

## Step 4 — Add your API Key

This is the most important step. Your API key must never go in the HTML file — Netlify stores it securely on the server.

1. In your Netlify dashboard, go to your site
2. Click **Site configuration** → **Environment variables**
3. Click **Add a variable**
4. Set:
   - **Key:** `ANTHROPIC_API_KEY`
   - **Value:** paste your key (`sk-ant-...`)
5. Click **Save**
6. Go to **Deploys** → click **Trigger deploy → Deploy site**
   *(The site needs to redeploy to pick up the new environment variable)*

---

## Step 5 — Test it

1. Open your new Netlify URL
2. Go to **My Cars → Add Vehicle**
3. Upload a photo of a car
4. You should see the scanning animation, then all fields auto-fill

---

## Troubleshooting

**Still getting "AI offline" error:**
- Check the API key is set correctly in Netlify environment variables (no spaces, no quotes)
- Make sure you triggered a redeploy after adding the key
- Open browser DevTools → Network tab → look for a failed `/api/claude` request and check the error

**"API key not configured" error:**
- The environment variable name must be exactly `ANTHROPIC_API_KEY` (all caps)

**Site shows old version:**
- Netlify should auto-deploy when you push to GitHub. Check the Deploys tab — look for a green "Published" status

**Want to keep GitHub Pages too:**
- That's fine. GitHub Pages will serve the app without AI (falls back to demo data). Your Netlify URL is the full-featured version.

---

## Going forward

Every time you push a change to GitHub, Netlify automatically rebuilds and redeploys. You never need to touch Netlify again — just keep using GitHub as normal.

---

## Your new URL structure

| What | Where |
|---|---|
| Your app | `https://your-site.netlify.app` |
| AI proxy | `https://your-site.netlify.app/api/claude` |
| Source code | Your GitHub repo (unchanged) |
| API key | Netlify environment variables (secure, server-side only) |
