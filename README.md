# CAPTCHA POC — Google reCAPTCHA v3 vs Cloudflare Turnstile

A side-by-side proof of concept for the **Sanus reCAPTCHA Research Story**.

---

## Running Locally

> ⚠️ **Do NOT open `index.html` directly** (via `file://`). Google reCAPTCHA blocks `file://` protocol and will show an "Invalid domain" error. Always serve via `localhost`.

### Option 1 — `npx serve` (recommended, no install)

```bash
# From the captcha-poc folder:
npx serve . -p 3456
```

Then open: [http://localhost:3456](http://localhost:3456)

### Option 2 — Python (if Node.js is not available)

```bash
# Python 3
python -m http.server 3456

# Python 2
python -m SimpleHTTPServer 3456
```

Then open: [http://localhost:3456](http://localhost:3456)

> 💡 Must have an internet connection — both CAPTCHA scripts load from CDN.

---

## Test Keys Used

### ✅ No account login required for this POC

Both providers publish **official public test keys** in their documentation specifically for local development and demos. These keys are hardcoded in `index.html` and work for anyone out of the box.

| Service | Type | Site Key | Source |
|---|---|---|---|
| Google reCAPTCHA v3 | Public test key | `6LeIxAcTAAAAAJcZVRqyHh71UMIEGNQ_MXjiZKhI` | [Google reCAPTCHA FAQ](https://developers.google.com/recaptcha/docs/faq#id-like-to-run-automated-tests-with-recaptcha-v2-how-should-i-do-this) |
| Cloudflare Turnstile | Public test key | `1x00000000000000000000AA` | [Cloudflare Turnstile Docs](https://developers.cloudflare.com/turnstile/reference/testing/) |

> ⚠️ These test keys **only work on `localhost`** (or `127.0.0.1`). They will not work on any public domain — that's by design.

### Cloudflare Turnstile — Additional Test Keys

Swap the `data-sitekey` value in `index.html` to simulate different bot scenarios:

| Site Key | Behavior |
|---|---|
| `1x00000000000000000000AA` | Always passes silently (default in this POC) |
| `2x00000000000000000000AB` | Always blocks — simulates bot detection |
| `3x00000000000000000000FF` | Forces a visible interactive challenge |

---

## Getting Real Production Keys

> **Account login required for real keys.** Do not use production keys in this POC.

### Google reCAPTCHA

1. Go to [https://www.google.com/recaptcha/admin](https://www.google.com/recaptcha/admin)
2. Sign in with a **Google account** (company Google Workspace account recommended)
3. Register a new site → choose **reCAPTCHA v3**
4. Add your domain(s) (e.g. `sanus.com`, `staging.sanus.com`)
5. You'll receive a **Site Key** (public) and **Secret Key** (server-side only)

### Cloudflare Turnstile

1. Go to [https://dash.cloudflare.com](https://dash.cloudflare.com) → **Turnstile** section
2. Sign in with a **Cloudflare account** (free tier is sufficient)
3. Add a new widget → choose **Managed** (invisible) mode
4. Add your domain(s)
5. You'll receive a **Site Key** (public) and **Secret Key** (server-side only)

> 💡 If Sanus/Legrand is already on Cloudflare (CDN/DDoS), the Turnstile dashboard is already accessible under the same account — no new signup needed.

---

## What the POC Demonstrates

### Google reCAPTCHA v3 (Left Panel)
- Score-based background check runs on page load
- On form submit: `grecaptcha.execute()` is called with action `"login"`
- Token is sent to (mocked) server → score returned
- Score `≥ 0.7` = low risk, score `< 0.5` = high risk (would trigger challenge in production)

### Cloudflare Turnstile (Right Panel)
- Invisible widget initializes **immediately on page load**
- Token is issued silently — **button is locked until token is ready**
- On form submit: token is attached to request body and (mock) server verifies
- User never sees or interacts with anything

---

## Replacing Mock with Real Server Verification

### Google reCAPTCHA — Server call
```http
POST https://www.google.com/recaptcha/api/siteverify
Content-Type: application/x-www-form-urlencoded

secret=YOUR_SECRET_KEY&response=TOKEN&remoteip=USER_IP
```

### Cloudflare Turnstile — Server call
```http
POST https://challenges.cloudflare.com/turnstile/v0/siteverify
Content-Type: application/json

{ "secret": "YOUR_SECRET_KEY", "response": "TOKEN", "remoteip": "USER_IP" }
```

### NextJS API Route (example)
```ts
// src/pages/api/verify-captcha.ts
export default async function handler(req, res) {
  const { token, provider } = req.body;
  const url = provider === 'turnstile'
    ? 'https://challenges.cloudflare.com/turnstile/v0/siteverify'
    : 'https://www.google.com/recaptcha/api/siteverify';

  const result = await fetch(url, {
    method: 'POST',
    body: JSON.stringify({ secret: process.env.CAPTCHA_SECRET_KEY, response: token }),
    headers: { 'Content-Type': 'application/json' }
  }).then(r => r.json());

  if (!result.success) return res.status(403).json({ error: 'Bot detected' });
  return res.status(200).json({ ok: true });
}
```

---

## Environment Variables (for real implementation)

```bash
# Add to .env.* files
NEXT_PUBLIC_TURNSTILE_SITE_KEY=<from Cloudflare dashboard>
TURNSTILE_SECRET_KEY=<from Cloudflare dashboard — server-side only>

NEXT_PUBLIC_RECAPTCHA_SITE_KEY=<from Google reCAPTCHA admin>
RECAPTCHA_SECRET_KEY=<from Google reCAPTCHA admin — server-side only>
```

---

## Related Files
- [`captcha_research.md`](./captcha_research.md) — Full research documentation (same folder)
- [`index.html`](./index.html) — This POC
