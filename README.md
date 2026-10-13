# ShopAssist + WARDEN — Whole Project Guide

AI shopping chatbot (refunds, cancellations, returns, order tracking) governed by
the WARDEN engine, usable from any Android phone. One repo map, how it runs,
and how to operate it.

## 1. The three pieces

| # | Directory | What it is | Git remote | Size |
|---|-----------|-----------|------------|------|
| 1 | `/home/deadw4lker/shopassist_app` | Android phone app (native chat UI) | — (not a repo) | ~9 MB |
| 2 | `/home/deadw4lker/agento` | ShopAssist chatbot (brain + APIs) | `deadw4lker/shopassist-api` | ~1 MB |
| 3 | `/home/deadw4lker/Downloads/CODEX-3.0-main` | WARDEN governance platform (engine + console) | `deadw4lker/warden_engine` | ~222 MB |

and warden.zip is our OPA file.

## 2. How it fits together (live right now)

```
Anyone's phone (ShopAssist APK)
  │  https://shopassist-api-z5wy.onrender.com  + app token (⚙ settings)
  ▼
Public agent — Render free tier (WARDEN mode)
  │  WARDEN_BASE_URL = Cloudflare tunnel (auto-revived by watchdog)
  ▼
Laptop WARDEN :8000  ←── governs every refund/cancel/return ──→ you watch :5173
  (tamper-evident ledger, approvals, kill switch, policies)
```

- The phone **never** holds the Warden key — only the low-privilege app token.
- Local-first alternative: point the app at `http://<laptop-ip>:8502` (same Wi-Fi) and skip Render entirely.

## 3. Run everything locally (laptop)

Prerequisites: Python 3.11+, Node 18+, Android SDK (only for APK builds).

```bash
# Terminal 1 — WARDEN engine (http://127.0.0.1:8000)
cd /home/deadw4lker/Downloads/CODEX-3.0-main/backend
python3 -m uvicorn app.main:app --host 127.0.0.1 --port 8000

# Terminal 2 — WARDEN console (http://127.0.0.1:5173)
cd /home/deadw4lker/Downloads/CODEX-3.0-main/frontend
npm run dev

# Terminal 3 — Chat API for phones (http://<laptop-ip>:8502)
cd /home/deadw4lker/agento
python3 -m uvicorn chat_api:app --host 0.0.0.0 --port 8502

# Terminal 4 (optional) — Streamlit test UI (http://<laptop-ip>:8501)
cd /home/deadw4lker/agento
python3 -m streamlit run app.py --server.address 0.0.0.0 --server.port 8501
```

Logins: console `admin / demo123`. Health checks: `/health` on `:8000` and `:8502`.

First-time setup: `pip install -r backend/requirements.txt` (engine),
`pip install -r requirements.txt` (agent), `npm install` (console).
Register the bot once: console → **Connect → Register agent** → `shopassist_bot`
(actions `issue_refund, issue_replacement, update_record, create_ticket, view_order`,
data `public, internal`, max `100000`) → put the issued `wrd_...` key in
`agento/.env` as `WARDEN_API_KEY`.

## 4. Phone app — build & install

```bash
cd /home/deadw4lker/shopassist_app
/tmp/opencode/gradle-8.7/bin/gradle assembleDebug --no-daemon
# APK: app/build/outputs/apk/debug/app-debug.apk
adb install -r app/build/outputs/apk/debug/app-debug.apk   # USB debugging ON
```

In-app: server URL defaults to the public agent; override in ⚙ if testing
locally. Pick a customer (CUST-01…CUST-06 dropdown). Paste the app token once
in ⚙ when the server requires one.

## 5. Public hosting (Render + Cloudflare tunnel, free tier)

- **Engine + console:** Render Blueprint from `warden_engine` (`render.yaml`):
  `warden-backend` (Docker) + `warden-console` (static, `VITE_API_BASE` wired).
- **Agent:** Render Blueprint from `shopassist-api` (`render.yaml`): `shopassist-api`.
- **Local engine exposure:** Cloudflare quick tunnel
  `~/.local/bin/cloudflared tunnel --url http://localhost:8000`
  guarded by `/tmp/opencode/cf-watch.sh` (auto-revives; logs URL changes loudly —
  a changed URL means updating `WARDEN_BASE_URL` on Render once).
- **Env secrets (Render dashboard, never in code):**
  `JWT_SECRET`, `KEY_ENCRYPTION_SECRET` (backend), `WARDEN_API_KEY` (agent —
  re-register bot after every free-tier wipe, see `RENDER.md` §4),
  `APP_API_KEY` (agent + phones), fresh `LLM_API_KEY` (rotate the demo one).
- Full deploy/runbook detail: `CODEX-3.0-main/RENDER.md`. Business onboarding:
  `CODEX-3.0-main/ONBOARDING.md`.

## 6. Daily operation

- **Watch:** console Activity log / Approvals / ledger. Big refunds arrive as
  `PENDING_APPROVAL` — approve/reject there.
- **Kill switch:** Agents → pause `shopassist_bot` → every user gets safe
  refusals instantly; resume when clear.
- **Abuse:** spam/repeat-flood/injection auto-blocks per customer
  (`abuse_guard.py`); blocks are reported to the Warden ledger.
- **Leaked key:** Connect → Key management → Rotate (old key dies instantly).
- **Laptop must stay on** while localhost is the engine (tunnel dies with it).
- **Free-tier realities:** cold starts (~50s first chat), SQLite wipes on
  restart (re-register bot per `RENDER.md` §4), 750h/month shared hours.

## 7. Troubleshooting

| Symptom | Cause → fix |
|---|---|
| App: connection error | Server URL/token in ⚙; laptop IP changed (`hostname -I`); Render service asleep (wait ~50s, retry) |
| App: "Service Unavailable" / 503 | Tunnel died → check `/tmp/opencode/cf-watch.log`; watchdog usually revives in ~1 min |
| Chat: "Missing/Invalid app token" | Paste matching `APP_API_KEY` in ⚙ |
| Chat: governance BLOCKED, "fail closed" | Warden unreachable (tunnel down) or bot key mismatch → check tunnel, re-register/rotate key |
| Console shows nothing from phone | Wrong console (local vs public are separate databases) or chat was read-only (`my orders` skips governance by design) |
| `npm run build` fails | Run inside `frontend/`; needs `VITE_API_BASE` set for public builds |
| APK build fails | `ANDROID_HOME`/`JAVA_HOME` (JDK 21) env; Gradle 8.7 at `/tmp/opencode/gradle-8.7` |

## 8. Key URLs (update if redeployed/renamed)

- Local: engine `:8000`, console `:5173`, chat API `:8502`, Streamlit `:8501`
- Public: `https://warden-backend-wuym.onrender.com`,
  `https://warden-console.onrender.com`,
  `https://shopassist-api-z5wy.onrender.com`
<img width="1904" height="1040" alt="image" src="https://github.com/user-attachments/assets/bb2e15bb-ca75-4f40-9b18-d8c03b6678b9" />

<img width="720" height="1640" alt="WhatsApp Image 2026-10-13 at 8 50 59 AM(1)" src="https://github.com/user-attachments/assets/43a8d726-7509-4d58-a051-b0893bd2c195" />


