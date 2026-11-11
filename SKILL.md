---
name: Rakazo Self-Host
description: "Install, configure, verify, maintain or troubleshoot a self-hosted Rakazo server (Docker published images + Tailscale HTTPS) on a Mac, including the \"Rakazo 自架\" companion app, Composio/OpenRouter keys, email (SMTP/Resend), first-account creation, upgrades and backups."
---

# Rakazo 自架（Self-Host）作業手冊

Use for any request about a self-hosted Rakazo server: first install, `.env` changes, "連線應用不能用", "登不進去", 寄信, 更新, 備份, 桌面 App 連線. Reply to the user in their language; keep secrets out of chat.

## 0. Ground rules
- Run commands on the **server Mac** via `user_computer_shell`. First confirm you are on the right machine: `hostname; tailscale status --json | python3 -c 'import json,sys;print(json.load(sys.stdin)["Self"]["DNSName"])'`. If two Macs have "Let bots use this computer" on, ask the user to turn it off on the wrong one.
- A shell call that reports "Not allowed" may still have executed. Re-check state (`ls ~/rakazo`, `docker compose -p rakazo ps`) before retrying; never reinstall blindly.
- Never print, echo or paste keys. Check keys by prefix (`${v:0:3}`) or HTTP status only. Pass secrets to curl with `-K -` via stdin.
- Before editing `~/rakazo/.env`, copy it to `.env.bak-YYYYmmdd-HHMMSS`. Only change the keys you need. After any change, restart: `cd ~/rakazo && docker compose --env-file .env -f docker-compose.images.yml -p rakazo up -d --pull never --wait`.
- If the user has the companion app (`/Applications/Rakazo 自架.app`, source `~/rakazo/tools/RakazoServer`, rebuild with `./build.sh`), point them to it first: 「總覽」 health check, 「設定」 save-and-apply, 「安裝」 6-step wizard. Manual: `~/rakazo/docs/自架操作手冊.md`.

## 1. Preflight (all must pass)
- Docker Engine ≥ 26 running (`docker version --format '{{.Server.Version}}'`), `docker compose version`, `curl`, `openssl`.
- Tailscale running and logged in; `CertDomains` non-empty and MagicDNS on (`tailscale status --json`). If not: Tailscale admin → DNS → enable MagicDNS + HTTPS Certificates.
- Pick a port. Do NOT take the Tailscale root URL if `tailscale serve status` already maps it to something else; use `:5173`. Never expose `3100`.
- ≥ 15 GB free disk. Host must not sleep.

## 2. Install (official published images)
```bash
mkdir -p ~/rakazo && cd ~/rakazo
curl -fsSLO https://raw.githubusercontent.com/elie222/rakazo/main/infra/compose/install-images.sh
bash install-images.sh --prepare-only      # creates .env with random secrets, does not start
```
Edit `.env` (same HTTPS origin in all three):
```env
BETTER_AUTH_URL=https://<host>.<tailnet>.ts.net:5173
WEB_ORIGIN=https://<host>.<tailnet>.ts.net:5173
API_URL=https://<host>.<tailnet>.ts.net:5173
RAKAZO_HOST=<host>.<tailnet>.ts.net
SIGNUP_ALLOWLIST=<owner email>
# optional: OPENROUTER_API_KEY, COMPOSIO_API_KEY, RAKAZO_WEB_PORT (if local port ≠ 5173)
```
Then `bash install-images.sh` (pulls + `up --wait`; rerunning keeps the existing `.env`), then
`tailscale serve --bg --https=5173 http://127.0.0.1:5173`. Verify `curl -s -o /dev/null -w '%{http_code}' https://<url>` returns 200.
Other hostnames (LAN IP, short name) go in `RAKAZO_ALLOWED_HOSTS`, or the web app rejects them and the computer screen stays blank.

## 3. First account — choose ONE path before signing up
With a non-empty `SIGNUP_ALLOWLIST`, sign-in requires a verified email (including the first account), and verification needs email delivery.
- **A (recommended):** configure email first (§4), then sign up on the web and click the verification link.
- **B (no email):** set `SIGNUPS_ENABLED=false`, restart, then create a verified account with the official tool (password ≥ 16 chars, passed by env, never on the command line):
```bash
docker compose --env-file .env -f docker-compose.images.yml -p rakazo exec -T -e OWNER_EMAIL -e OWNER_PASSWORD api node dist/cli/bootstrap-owner.js
```
Symptom of getting this wrong: login fails after a restart with `EMAIL_NOT_VERIFIED` in api logs. Fix by path A or B; do not edit the database unless the user explicitly accepts a non-official fix.

## 4. Email
- Gmail (simplest): enable 2-Step Verification, create an App Password (16 letters) on the **sending** account, then
  `EMAIL_PROVIDER=smtp`, `SMTP_URL=smtps://<user%40gmail.com>:<app-password>@smtp.gmail.com:465`, `EMAIL_FROM=Rakazo <user@gmail.com>`. Sender and recipient may be the same Gmail. Workspace admins may disable app passwords.
- Resend: verified domain + sending key (`re_…`): `EMAIL_PROVIDER=resend`, `EMAIL_API_KEY`, `EMAIL_FROM=Rakazo <noreply@domain>`.
- `smtps://` = port 465 implicit TLS; `smtp://` = 587 STARTTLS. URL-encode `@ : /` in credentials. Always send a test mail before applying.

## 5. Keys
- **Composio:** must be the **Platform project API key (`ak_…`)** from dashboard.composio.dev → PLATFORM → API Keys. The `ck_…` key on the "Connect" settings page is a personal MCP key and returns `401 Invalid API key`. Composio's onboarding "install the skill" step is for coding agents; skip it. Check: `curl -s -o /dev/null -w '%{http_code}' -H "x-api-key: $KEY" 'https://backend.composio.dev/api/v3/auth_configs?limit=1'` → 200.
- **OpenRouter:** `curl ... -H "Authorization: Bearer $KEY" https://openrouter.ai/api/v1/key` → 200. Models can also be connected in the web UI after signup.
- After changing keys, restart and confirm the containers see them: `docker compose -p rakazo exec -T api sh -c 'printf %s "$COMPOSIO_API_KEY" | cut -c1-3'`.

## 6. Clients
- Browser: open the URL. Desktop app: Cmd+Shift+K → **Existing instance** → URL. **"This computer" installs a separate stack** (`rakazo-desktop-*` containers, port 45173) — not the self-hosted server. Mobile: "Use a custom server" with Tailscale on.
- To let bots use a Mac: desktop app on that Mac → Settings → General → Computers → "Let bots use this computer".
- Passkeys are hidden for self-hosted servers in the desktop app; use passwords.
- Self-hosted accounts, bots and chats are separate from rakazo.com.

## 7. Operate
- Health: `docker compose -p rakazo ps` (postgres, supervisor, api, worker, web running/healthy), `curl` local `http://127.0.0.1:5173` and the Tailscale URL, `tailscale serve status`, key checks above.
- Logs: `docker compose --env-file .env -f docker-compose.images.yml -p rakazo logs --tail 200 api worker`.
- Diagnostics (no secrets): `... exec -T api node dist/cli/diagnostics.js`.
- Backup: `... exec -T postgres sh -c 'pg_dump -Fc -U "$POSTGRES_USER" "$POSTGRES_DB"' > backups/x.dump` **plus a copy of `.env`** (ENCRYPTION_KEY is needed to read stored credentials). This does not include bot home files.
- Upgrade: back up, then `... pull` and `... up -d --pull never --wait`. `RAKAZO_IMAGE_TAG=edge` tracks main; pin a release tag (e.g. `vX.Y.Z`) for stability. The images file stays on Postgres 16.
- Remove an accidental desktop "This computer" stack: stop/remove only `rakazo-desktop-*` containers, their volumes/networks and `~/Library/Application Support/Rakazo/stack`; never touch the `rakazo` project or shared `postgres:16` image.

## 8. Quick triage
| Symptom | Likely cause | Fix |
|---|---|---|
| Connected apps all fail | `ck_` or revoked Composio key | New `ak_` Platform key, restart |
| `EMAIL_NOT_VERIFIED` after restart | allowlist set, no email | §3 path A or B |
| URL won't open from other devices | serve mapping missing / Tailscale off | `tailscale serve --bg --https=PORT http://127.0.0.1:PORT` |
| Blank computer screen | Host header not allowed | add name to `RAKAZO_ALLOWED_HOSTS` |
| `.env` change ignored | not restarted | `up -d --pull never --wait` |
| Bot ran commands on the wrong Mac | two Macs exposed | turn off "Let bots use this computer" on the other |

Official docs: https://rakazo.com/docs/self-host.md and linked pages (accounts, clients, integrations, upgrading, backups). Fetch them when behaviour seems to differ — they change.
