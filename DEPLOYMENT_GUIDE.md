# Quest for Compliances — Deployment Guide

`index.html` is a **single self-contained file** — no build step, no `npm install`, no backend
required just to view it. Every calculator, the law library, the TDS/penalty reckoners and the
GST workbench run fully in the browser.

There is **one exception**: the AI Consultant. It calls Anthropic's API directly, which works
inside the Claude artifact preview but **not** on a normal static host (a browser can't safely
hold an API key). Part 2 below fixes that with a tiny free proxy.

---

## Part 1 — Publish the site (free)

Pick any one. All are free and give you HTTPS.

### Option A — GitHub Pages (you already use this)

```bash
# from a fresh folder containing index.html
git init
git add index.html
git commit -m "Quest for Compliances"
git branch -M main
git remote add origin https://github.com/<your-username>/quest-for-compliances.git
git push -u origin main
```

Then on GitHub: **Settings → Pages → Build and deployment → Source: Deploy from a branch →
`main` / `root` → Save.** Live in ~1 minute at
`https://<your-username>.github.io/quest-for-compliances/`.

> Tip: if you want it at the repo root URL with no subfolder, name the repo
> `<your-username>.github.io`.

### Option B — Netlify (drag & drop, no git)

Go to **app.netlify.com → Add new site → Deploy manually**, then drag the folder containing
`index.html` onto the page. Done. You get a `*.netlify.app` URL instantly; add a custom domain
later if you like.

### Option C — Cloudflare Pages

**dash.cloudflare.com → Workers & Pages → Create → Pages → Upload assets.** Upload `index.html`.
Free, fast global CDN, and it's the same platform as the AI proxy in Part 2 (so everything lives
in one place).

### Option D — Vercel

```bash
npm i -g vercel
vercel        # answer the prompts; framework = "Other"
```

---

## Part 2 — Make the AI Consultant work live

**Why it needs this:** the consultant calls `https://api.anthropic.com/v1/messages`. That call
needs your secret API key. If you put the key in `index.html`, anyone can read it and run up your
bill. The fix is a **proxy**: a tiny server-side endpoint that holds the key, receives the
browser's request, and forwards it to Anthropic. The browser never sees the key.

Pick **one** of the three below.

### Proxy A — Cloudflare Worker (recommended: free, 5 minutes)

1. **dash.cloudflare.com → Workers & Pages → Create → Create Worker.** Name it e.g. `qfc-ai`.
2. Replace the worker code with:

```js
export default {
  async fetch(request, env) {
    // CORS preflight
    if (request.method === "OPTIONS") {
      return new Response(null, { headers: cors() });
    }
    if (request.method !== "POST") {
      return new Response("POST only", { status: 405, headers: cors() });
    }
    const body = await request.text();
    const r = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "x-api-key": env.ANTHROPIC_API_KEY,      // set as a secret, see step 3
        "anthropic-version": "2023-06-01",
      },
      body, // forward the browser's payload (model, system, messages, max_tokens) unchanged
    });
    const text = await r.text();
    return new Response(text, { status: r.status, headers: cors() });
  },
};

function cors() {
  return {
    "Access-Control-Allow-Origin": "*",        // tighten to your domain in production
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type",
    "Content-Type": "application/json",
  };
}
```

3. **Add your key as a secret** (never in code): Worker → **Settings → Variables and Secrets →
   Add → "Secret"** → name `ANTHROPIC_API_KEY`, value = your key from
   `console.anthropic.com`. Deploy.
4. Copy your Worker URL, e.g. `https://qfc-ai.<your-subdomain>.workers.dev`.
5. **One edit in `index.html`** — find this line inside `sendChat()`:

   ```js
   const resp=await fetch("https://api.anthropic.com/v1/messages",{
   ```

   change it to your Worker URL:

   ```js
   const resp=await fetch("https://qfc-ai.<your-subdomain>.workers.dev",{
   ```

   Re-deploy the site. The consultant now works on any host.

### Proxy B — Netlify Function (if you host on Netlify)

Create `netlify/functions/chat.js`:

```js
exports.handler = async (event) => {
  if (event.httpMethod !== "POST") return { statusCode: 405, body: "POST only" };
  const r = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": process.env.ANTHROPIC_API_KEY,
      "anthropic-version": "2023-06-01",
    },
    body: event.body,
  });
  return { statusCode: r.status, body: await r.text() };
};
```

Set `ANTHROPIC_API_KEY` under **Site settings → Environment variables**. Then in `index.html`
point the fetch at `"/.netlify/functions/chat"`. (Vercel is identical with a file at
`api/chat.js` and a fetch to `"/api/chat"`.)

### Proxy C — Your existing Express backend (port 3001)

You already run an Express server. Add one route:

```js
// in your pratyaksha-backend app
const express = require("express");
const cors = require("cors");
app.use(cors());
app.use(express.json({ limit: "1mb" }));

app.post("/api/chat", async (req, res) => {
  const r = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": process.env.ANTHROPIC_API_KEY, // keep in a .env file, never commit it
      "anthropic-version": "2023-06-01",
    },
    body: JSON.stringify(req.body),
  });
  res.status(r.status).send(await r.text());
});
```

In `index.html` point the fetch at `"http://localhost:3001/api/chat"` (local) or your deployed
backend URL. Node 18+ has `fetch` built in; on older Node add `node-fetch`.

---

## Golden rule

**The API key lives only on the server side (Worker secret / env variable / `.env`).**
It must never appear in `index.html` or anything you push to a public GitHub repo. The proxy
exists precisely so the key stays secret while the browser still gets answers.

---

## What's "full" vs "seeded" in the library

Six domains ship with rich provisions + landmark case law: **Companies Act 2013, SEBI,
Income-tax Act 2025, GST, IBC, Contract Act.** The other ~15 are *seeded* — statute map, an
anchor provision/case, and a roadmap note — with the **AI consultant covering any provision in
those Acts on demand.** To promote a seeded domain to full, just add objects to that law's
`provisions` array in the `LAWS` data block (each takes a section, a plain-language summary, and
optional `cases`). The UI grows automatically — no other change needed.
