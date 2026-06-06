# mipus / Xpat Lookup — Project Knowledge Export

> **Purpose:** Feed this document to another AI agent or project so it inherits full context from the mipus development conversation (May 2026).  
> **Repo:** https://github.com/adhuhaam/mipus  
> **Production:** https://workpermit-mv.vercel.app  
> **Reference OCR UX:** https://github.com/adhuhaam/xxpat.git (browser Tesseract; uses HTML scrape, not official API)

---

## 1. Product summary

**Unofficial** Maldives work permit lookup tool — same data as the official **Xpat MV** mobile app.

| Channel | Input | Output |
|---------|--------|--------|
| **Web PWA** | Manual WP + passport, or camera/upload image | Full employee profile UI, photo, official permit card PNG |
| **Telegram bot** | Two lines of text, or photo of document | Structured HTML message + **permit card image only** (no employee photo as of PR #12) |

Both channels require **work permit number** and **passport number** for API lookup (single field → 400 from upstream).

---

## 2. Official API (must use for mipus)

Discovered from reverse-engineering `Xpat MV.xapk` (gitignored locally; was briefly committed then removed).

| Item | Value |
|------|--------|
| **Base URL** | `https://mobile-xpat.egov.mv/api/v1` |
| **Auth header** | `ApiKey: <key>` (NOT `Authorization`) |
| **Env var name** | `XPAT_API_KEY` |
| **Swagger** | `https://mobile-xpat.egov.mv/swagger/v1/swagger.json` |

### Endpoints (only these exist)

| Method | Path | Query params | Returns |
|--------|------|--------------|---------|
| GET | `/WorkPermit` | `WorkPermitNumber`, `PassportNumber` | JSON — all employee/employer fields |
| GET | `/WorkPermit/GetImage` | `PhotoId`, `ServiceId` | Employee photo (from `photoUrl` in JSON) |
| GET | `/WorkPermitCard/GetWorkPermitCard` | `WorkPermitNumber`, `PassportNumber` | Permit card PNG |

**Test pair (known working):** `WP00595305` / `V7255877`

**Do not** use xxpat’s scrape endpoint (`xpat.egov.mv/.../WorkPermitVerify`) unless explicitly building a name-based fallback — mipus uses official API only.

---

## 3. Tech stack

| Layer | Choice |
|-------|--------|
| Framework | Next.js 15 App Router, TypeScript |
| Hosting | Vercel (needs server for API proxy — GitHub Pages rejected) |
| Styling | Custom CSS (dark gray `#2b2b2b`, accent blue `#3b82f6`) |
| PWA | `manifest.webmanifest`, `sw.js` v2 (network-first HTML; never cache-first `/_next`) |
| OCR web | **Tesseract.js v7 in browser** (`lib/ocr-scan-browser.ts`) — xxpat pattern |
| OCR Telegram | **Tesseract.js v7 on server** (`lib/ocr-scan-server.ts`) |
| Image preprocess (server) | `sharp` — rotate + resize only |
| Telegram | Webhook (not long polling), HTML `parse_mode` |

### Dependencies (main)

```
next, react, react-dom, sharp, tesseract.js@^7.0.0
```

### Rejected approaches (lessons learned)

| Approach | Why rejected |
|----------|----------------|
| **PaddleOCR / @gutenye/ocr-node** | Vercel lambda >250MB; ONNX bundling pain |
| **Server OCR for web PWA** | Slow, WASM ENOENT, timeouts; moved to browser |
| **GitHub Pages** | No server for API key proxy |
| **xxpat HTML scrape** | Different data source; name OR passport; no card API |

---

## 4. Architecture & key files

```
app/
  page.tsx                    → LookupApp
  api/work-permit/route.ts    → proxy WorkPermit JSON
  api/work-permit/photo/      → proxy GetImage
  api/work-permit/card/       → proxy GetWorkPermitCard
  api/ocr/route.ts            → optional server OCR (web uses browser now)
  api/telegram/webhook/       → bot updates
  api/telegram/setup/         → register webhook + setMyCommands

components/
  LookupApp.tsx               → form + scan + results
  DocumentScan.tsx            → camera/upload → browser OCR
  EmployeeProfile.tsx         → web results UI

lib/
  xpat-api.ts                 → upstream URLs + ApiKey header
  xpat-lookup.ts              → shared lookup logic
  ocr-extract.ts              → WP/passport regex (xxpat + labels)
  ocr-scan-browser.ts         → Tesseract v7 client
  ocr-scan-server.ts          → Tesseract v7 Telegram photos
  ocr-preprocess.ts           → sharp for server
  telegram-handler.ts         → text / photo routing
  telegram-process-lookup.ts    → lookup → messages → card image
  telegram-api.ts             → sendMessage, sendPhoto, download, webhook secret
  format-telegram-message.ts  → sectioned HTML reply
  parse-bot-message.ts        → HELP_TEXT, two-line parse

scripts/telegram-set-webhook.mjs

vercel.json                   → webhook maxDuration 60s, 512MB
next.config.ts                → outputFileTracingIncludes WASM for /api/telegram/webhook only
```

### Data flow

```
User input (text or image)
  → [image] OCR extract WP + passport
  → GET /api/work-permit (proxied)
  → Web: EmployeeProfile | Bot: HTML sections + permit card PNG
```

---

## 5. Environment variables (Vercel)

| Variable | Required | Notes |
|----------|----------|--------|
| `XPAT_API_KEY` | Yes | Same key as mobile app |
| `TELEGRAM_BOT_TOKEN` | Bot | From BotFather; **never commit**; rotate if leaked in chat |
| `TELEGRAM_WEBHOOK_SECRET` | Recommended | Random string you invent; must match `setWebhook` |
| `SETUP_SECRET` | Optional | Protects `POST /api/telegram/setup` |
| `WEBHOOK_URL` | **Not read at runtime** | Only for local `telegram-set-webhook.mjs` |

---

## 6. Telegram bot — critical setup

Creating a bot in BotFather is **not enough**. Must register webhook:

```bash
TELEGRAM_BOT_TOKEN='token-no-brackets' \
WEBHOOK_URL='https://workpermit-mv.vercel.app/api/telegram/webhook' \
TELEGRAM_WEBHOOK_SECRET='same-as-vercel' \
node scripts/telegram-set-webhook.mjs
```

Script also calls `setMyCommands` for `/help` and `/start`.

**Alternative:** `POST /api/telegram/setup` with header `x-setup-secret: <SETUP_SECRET>`.

**Health check:** `GET /api/telegram/webhook` → `{"ok":true,"bot":true,"secret":true}`

**Common failures:**

| Symptom | Cause |
|---------|--------|
| Bot silent | Webhook never registered |
| Vercel 401 | `TELEGRAM_WEBHOOK_SECRET` mismatch |
| curl 404 on getWebhookInfo | Angle brackets around token in URL |
| Photo scan timeout | First OCR downloads tessdata; need >10s on Vercel |

### Help message (bilingual)

Defined in `lib/parse-bot-message.ts` — English + Bengali instructions, examples `WP00000000` / `A0000000`.

Reply keyboard on help: buttons `/help` | `/start`.

BotFather **description** (“What can this bot do?”) is separate — user configures manually.

### Telegram lookup reply format (HTML)

Sections: PERSONAL INFORMATION, WORK PERMIT DETAILS, EMPLOYMENT INFORMATION, VALIDITY DETAILS, VERIFICATION (eGov link).

Then: **permit card image only** (employee photo removed per user request, PR #12).

---

## 7. OCR details

### Extraction (`lib/ocr-extract.ts`)

- xxpat patterns: `WP\s*[-:]?\s*(\d{5,})`, passport `(?:passport|pp)...`, `\b([A-Z]\d{6,9})\b`
- Label patterns: WORK PERMIT NO, PASSPORT NO
- `fixOcrChars`: O→0, I/l→1

### Web

`Tesseract.recognize(file, 'eng', { logger })` in browser — progress UI in DocumentScan.

### Telegram

Server `recognize()` with tessdata from `https://tessdata.projectnaptha.com/4.0.0_fast`, cache `/tmp`.

### Vercel WASM

`next.config.ts` → `outputFileTracingIncludes` for `/api/telegram/webhook` only (~78MB with WASM). Without this: `ENOENT tesseract-core-simd.wasm`.

---

## 8. PWA / deploy gotchas

- **ChunkLoadError after deploy:** Old service worker cached HTML pointing at removed `_next` chunks. Fixed in PR #7: SW v2 network-first for HTML and `/_next`.
- **Lambda size:** Paddle/ONNX exceeded 250MB; Tesseract webhook ~78MB OK.
- **favicon:** redirect `/favicon.ico` → `/icon.svg`
- **`.gitignore`:** `*.apk`, `*.xapk`, `xapk_extracted/`

---

## 9. UI / UX decisions

- Web: employee profile hero, sections (permit, personal, employer), permit card image
- No footer raw API dump
- Theme: dark gray + blue accent
- Telegram: user-approved formal bilingual instructions

---

## 10. PR / merge history (GitHub)

| PR | Topic |
|----|--------|
| #5–#6 | Initial PWA, XAPK removal, vercel.json |
| #7 | PWA ChunkLoadError / service worker |
| #8 | Tesseract replaces PaddleOCR |
| #9 | WASM file tracing fix |
| #10 | Browser OCR (xxpat), rich Telegram replies, setup API |
| #11 | Bilingual help + keyboard buttons |
| #12 | Telegram: no employee photo, card only |

---

## 11. Related project: xxpat

| | xxpat | mipus |
|--|-------|-------|
| Lookup | HTML scrape `xpat.egov.mv` | Official mobile API |
| Second field | Name **or** passport | Passport **only** |
| OCR | Browser Tesseract v7 | Browser (web) + server (Telegram) |
| Telegram | None | Full bot |
| Deploy | Vercel | Vercel |

When porting UX from xxpat, take **OCR + scan UI patterns**, not the scrape API.

---

## 12. Security notes

- Never commit `TELEGRAM_BOT_TOKEN` or post in chat — revoke in BotFather if leaked
- `XPAT_API_KEY` server-side only (API routes)
- `TELEGRAM_WEBHOOK_SECRET` is user-generated, not from BotFather
- Unofficial tool — authorized use only

---

## 13. Commands for agents

```bash
npm install
cp .env.example .env.local   # set XPAT_API_KEY
npm run dev                  # http://localhost:3000
npm run build
npm run lint
```

Cloud agent branch naming (if applicable): `cursor/<name>-e9cc`

---

## 14. User intent (conversation arc)

1. Set up dev environment from empty repo
2. Extract API from `Xpat MV.xapk`
3. Build Vercel PWA like official app (lookup, photo, card)
4. Add Telegram bot (text + OCR)
5. Polish UI (employee profile, dark theme)
6. Fix OCR speed/quality → xxpat browser pattern
7. Fix Vercel deploy (lambda size, WASM, chunk cache)
8. Fix bot (webhook registration, env vars)
9. Rich Telegram message format (user template)
10. Bilingual help + /help /start buttons
11. Remove employee photo from Telegram replies

---

## 15. How to use this in another project

1. Attach this file to the new agent’s context (or copy into `AGENTS.md` / `docs/` there).
2. Point the agent at `github.com/adhuhaam/mipus` `main` for source of truth.
3. If reimplementing: keep **official API + dual OCR architecture** unless requirements change.
4. Always document **webhook registration** for any Telegram integration.

---

*Generated from Cursor agent session context. Last aligned with `origin/main` through PR #12 merge.*
