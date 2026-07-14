# Wiring up CLAIR's real brain (Cloudflare Worker + Gemini)

CLAIR already works offline with a scripted knowledge base. To switch her to a
**real LLM**, deploy the tiny proxy in `clair-worker.js` to Cloudflare and point
the page at it. The proxy holds your Gemini API key server-side — it never
lives in the website, and I never see it.

**Cost:** the worker uses Gemini 2.5 Flash, which has a **free tier** in
Google AI Studio — no credit card required to get started. A recruiter chat
uses a small fraction of the daily free quota.

---

## 1. Get a Gemini API key
1. Go to https://aistudio.google.com/apikey.
2. Sign in with a Google account, click **Create API key**.
3. Copy the key. You'll paste it into Cloudflare in step 3 — do **not** put it
   in any file.

## 2. Create the Worker on Cloudflare
Easiest path (no tools to install):
1. Sign up / log in at https://dash.cloudflare.com (free).
2. **Workers & Pages → Create → Create Worker.** Name it `clair`. Deploy the
   starter, then click **Edit code**.
3. Delete the sample code, paste the entire contents of `clair-worker.js`, and
   **Deploy**.
4. Copy your worker URL — it looks like `https://clair.<your-subdomain>.workers.dev`.

## 3. Add your API key as a secret
1. On the worker page: **Settings → Variables and Secrets → Add.**
2. Type = **Secret**, Name = `GEMINI_API_KEY`, Value = your Gemini key from step 1.
3. Save. (As a secret, it's encrypted and write-only — nobody can read it back.)

## 4. Allow your site's origin
Open `clair-worker.js` in the Cloudflare editor and check `ALLOWED_ORIGINS`
near the top. It already lists `https://chandlerpc.github.io` (your Pages site)
and localhost for testing. If you later add a custom domain, add it here and
re-deploy.

## 5. Point the page at the worker
In `clair-prototype.html` (and, once integrated, `index.html`), find:

```js
var CLAIR_ENDPOINT = "";        // e.g. "https://clair.your-name.workers.dev"
```

Set it to your worker URL from step 2:

```js
var CLAIR_ENDPOINT = "https://clair.your-subdomain.workers.dev";
```

Reload. CLAIR now answers with the real model. If the worker is ever
unreachable, she automatically falls back to the scripted answers, so she never
goes silent.

---

## What the worker does for you
- Keeps the API key server-side (never in the browser).
- Restricts calls to your own site's origin (CORS allowlist).
- Pins CLAIR's persona + everything about you into the system prompt.
- Caps reply length and conversation history to control cost and latency.
- Refuses off-topic / jailbreak attempts and won't reveal its instructions.

## Optional hardening (later)
- In Cloudflare, add a **Rate Limiting rule** on the worker route to blunt abuse.
- Add **Cloudflare Turnstile** (free CAPTCHA) if you ever see bot traffic.
- Edit the `SYSTEM_PROMPT` in `clair-worker.js` any time you want to change what
  she knows or how she talks, then re-deploy.
- Google AI Studio's free tier has daily request limits. If CLAIR ever gets
  heavy traffic, check https://ai.google.dev/pricing for paid-tier limits.
