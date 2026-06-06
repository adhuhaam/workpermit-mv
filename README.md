# Xpat Lookup PWA

Look up Maldives work permits (same data as **Xpat MV**): employee details, photo, and permit card — on the **web** or via **Telegram**.

## How it works

1. **Input** — type **work permit + passport**, or upload / photograph a document with both numbers
2. **OCR** (images) — **browser** Tesseract v7 on the web app; **server** Tesseract for Telegram photos
3. **Lookup** — official `mobile-xpat.egov.mv` API (both numbers required)
4. **Output** — profile on the website, or structured Telegram message + permit card (bot)

## Features

- Manual lookup form
- Camera / gallery scan (OCR in browser — no server upload on web)
- Telegram bot: two lines of text or a photo
- Rich Telegram reply (personal, permit, employment, validity, eGov link)

## Local development

```bash
npm install
cp .env.example .env.local
# XPAT_API_KEY required; TELEGRAM_* optional for bot
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

## Deploy to Vercel

| Variable | Required | Notes |
|----------|----------|--------|
| `XPAT_API_KEY` | Yes | Xpat Mobile API key |
| `TELEGRAM_BOT_TOKEN` | For bot | From [@BotFather](https://t.me/BotFather) |
| `TELEGRAM_WEBHOOK_SECRET` | Recommended | Random string; must match webhook registration |
| `SETUP_SECRET` | Optional | Protects `POST /api/telegram/setup` |

`vercel.json` sets **60s** timeout on the Telegram webhook (photo OCR on server).

### Bot not responding?

Creating a bot in BotFather is **not enough**. You must **register the webhook** once after deploy.

**Option A — local script** (from repo root):

```bash
TELEGRAM_BOT_TOKEN='your-token' \
WEBHOOK_URL='https://workpermit-mv.vercel.app/api/telegram/webhook' \
TELEGRAM_WEBHOOK_SECRET='same-as-vercel' \
node scripts/telegram-set-webhook.mjs
```

Expect `"ok": true`. Do **not** use angle brackets in URLs or tokens.

**Option B — Vercel setup route** (set `SETUP_SECRET` on Vercel, redeploy):

```bash
curl -X POST 'https://workpermit-mv.vercel.app/api/telegram/setup' \
  -H 'x-setup-secret: YOUR_SETUP_SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://workpermit-mv.vercel.app/api/telegram/webhook"}'
```

**Checks:**

1. `GET https://workpermit-mv.vercel.app/api/telegram/webhook` → `{"ok":true,"bot":true,"secret":true}`
2. `curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"` → `url` matches your app; no `last_error_message`
3. Message bot with two lines (no OCR): `WP00595305` then `V7255877`

| Symptom | Fix |
|---------|-----|
| Bot silent | Run webhook registration; check token on Vercel |
| Vercel log `401` | `TELEGRAM_WEBHOOK_SECRET` ≠ Telegram `secret_token` — re-run setWebhook |
| `getWebhookInfo` 404 | Token wrong; remove `<` `>` from curl URL |
| Photo scan slow/fails | First scan downloads OCR data (~15–30s); check Vercel logs |

**Security:** Never commit bot tokens. Revoke in BotFather if leaked.

### Test bot

```
WP00595305
V7255877
```

Then try a clear permit/passport photo.

## API routes

| Route | Purpose |
|-------|---------|
| `GET /api/work-permit` | Permit JSON |
| `GET /api/work-permit/photo` | Employee photo |
| `GET /api/work-permit/card` | Permit card PNG |
| `POST /api/telegram/webhook` | Bot updates |
| `POST /api/telegram/setup` | Register webhook (needs `SETUP_SECRET`) |
| `POST /api/ocr` | Optional server OCR (Telegram uses server OCR internally) |

## Agent / handoff docs

See **[docs/PROJECT_KNOWLEDGE.md](docs/PROJECT_KNOWLEDGE.md)** for a full export of project context (API, architecture, Telegram setup, OCR, deploy gotchas) to feed another AI agent or repo.

## Disclaimer

Unofficial tool. Use only for permits you are authorized to view.
