# ONMA Chatbot — Deployment Documentation

**Date:** October 1, 2026
**Migrated from:** Render (free tier, web service + PostgreSQL)
**Migrated to:** 1blu VPS RX16 (self-managed)

---

## 1. Server Details

| Item | Value |
|---|---|
| Hosting provider | 1blu |
| Plan | VPS RX16 (Virtuozzo) |
| OS | Ubuntu 24.04 LTS |
| Server hostname | v77134.1blu.de |
| Server IP | 195.90.220.251 |
| Control panel | Plesk (trial license — see Section 7) |
| Root SSH access | Yes (full root) |

## 2. Domain & DNS

| Item | Value |
|---|---|
| Parent domain | onmascout.de (hosted on STRATO, untouched) |
| Chatbot subdomain | chatbot.onmascout.de |
| Subdomain created via | STRATO panel → Domains → onmascout.de → Webserver tab → "Subdomain anlegen" |
| A record | chatbot.onmascout.de → 195.90.220.251 (set in STRATO's DNS tab for the subdomain) |
| SSL | Let's Encrypt (free), issued via Plesk extension |

**Important:** The main `onmascout.de` site and its ~1026 other subdomains remain fully on STRATO at `81.169.207.50`, completely unaffected by this migration. Only `chatbot.onmascout.de` points to the new 1blu VPS.

## 3. Plesk Setup

- Plesk installed via official one-click installer script
- Python support added via the **Python extension** (Extensions Catalog → "Python", not the deprecated `mod_python`)
- Node.js, Laravel, Git, SEO, and other extensions also available but unused for this project

### Python app configuration (Websites & Domains → chatbot.onmascout.de → Python)
| Setting | Value |
|---|---|
| Application root | `/chatbot` |
| Document root | `/chatbot/public` |
| Startup file | `passenger_wsgi.py` |
| Python version | 3.12.3 |
| Python path | `/var/www/vhosts/chatbot.onmascout.de/chatbot/.venv/bin/python` |

**Real filesystem path:** `/var/www/vhosts/chatbot.onmascout.de/chatbot/`

## 4. Code Deployment

- Deployed via Plesk's **Git** extension, pulling directly from the project's GitHub repository
- Deployed to `/chatbot` (not `/chatbot/public`) — the Python app root, so `manage.py`, `passenger_wsgi.py`, settings, etc. are **not** inside the publicly-served document root
- Virtual environment created via Plesk's "Create Virtual Environment" button (required `apt install python3.12-venv -y` first, since the system Python lacked the `venv`/`ensurepip` module)
- Dependencies installed via Plesk's "Install Dependencies" button, reading `requirements.txt`

## 5. Database — PostgreSQL

### Installed
- PostgreSQL 16 (via `apt install postgresql postgresql-contrib`)
- Running as a systemd service (`postgresql.service`), enabled on boot

### Database & user
| Item | Value |
|---|---|
| Database name | `onma_chatbot` |
| Database user | `onma_user` |
| Password | `Onma@#56ChatbotSecure67` |
| Host | `localhost` |
| Port | `5432` |

**Note on permissions:** PostgreSQL 15+ restricts schema creation to the database owner by default. Had to explicitly run:
```sql
ALTER SCHEMA public OWNER TO onma_user;
GRANT ALL ON SCHEMA public TO onma_user;
ALTER DATABASE onma_chatbot OWNER TO onma_user;
```

### Data migration from Render
- Render's PostgreSQL was running **version 18** — newer than the server's default client (v16), so `pg_dump` initially refused to connect (version mismatch).
- Fixed by adding the official PostgreSQL APT repo (PGDG) and installing `postgresql-client-18` specifically for the dump step:
  ```bash
  apt install -y postgresql-common
  /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y
  apt install -y postgresql-client-18
  ```
- Dump and restore commands used:
  ```bash
  /usr/lib/postgresql/18/bin/pg_dump --no-owner --no-acl "<render-connection-string>" -F c -f /root/render_backup.dump
  pg_restore --no-owner --no-acl -h localhost -U onma_user -d onma_chatbot /root/render_backup.dump
  ```
- All 17 tables successfully restored: `auth_*`, `bot_*` (answercache, botsettings, chatmessage, chatsession, knowledgechunk, knowledgeentry, lead), `django_*` (admin_log, content_type, migrations, session)

## 6. Environment Variables

Configured via a `.env` file at `/chatbot/.env` (loaded in `settings.py` via `python-dotenv`'s `load_dotenv()`), **not** Plesk's one-by-one environment variable UI (too slow for 30+ vars).

Key variables changed from the Render version:

| Variable | Render value | New (VPS) value | Reason |
|---|---|---|---|
| `DB_HOST` | Render's external host | `localhost` | DB now runs on the same server |
| `DB_USER` | `onma_chatbot` | `onma_user` | Matches the locally-created user |
| `DB_PASSWORD` | Render's password | `Onma@#56ChatbotSecure67` | New local DB password |
| `DJANGO_ALLOWED_HOSTS` | (was a full URL — invalid) | `chatbot.onmascout.de,www.chatbot.onmascout.de,195.90.220.251` | Must be bare hostnames |
| `DJANGO_DEBUG` | `True` | `False` | Required for production — hides stack traces/secrets |
| `SECURE_SSL_REDIRECT` | (unset) | `True` | Safe now that SSL is active |
| `SESSION_COOKIE_SECURE` | (unset) | `True` | Safe now that SSL is active |
| `CSRF_COOKIE_SECURE` | (unset) | `True` | Safe now that SSL is active |

All other variables (LLM_API_KEY, LLM_MODEL, EMBEDDING_*, RAG_*, email settings, Turnstile keys) were carried over unchanged from Render.

**⚠️ Known bugs fixed in `settings.py` during deployment:**
1. `ALLOWED_HOSTS` originally called `os.environ.get()` with 4 arguments instead of 2 — fixed to pass a single comma-separated default string.
2. `CORS_ALLOWED_ORIGINS` had a trailing comma inside a list comprehension, causing a `SyntaxError` — removed.

## 7. Security Verification

Confirmed the following are **not** publicly accessible (return 403/404, not file contents):
- `/.env`
- `/manage.py`
- `/passenger_wsgi.py`
- `/requirements.txt`
- `/.git/config`

This works because the Document root (`/chatbot/public`) is separate from the Application root (`/chatbot`) where the actual code and secrets live — all requests route through Passenger/Django rather than being served as static files.

## 8. Automation — Cron Jobs

Set up via `crontab -e` (root user):

```cron
0 2 * * * /root/backup_onma_chatbot.sh >> /var/log/onma_backup.log 2>&1
0 3 * * 0 cd /var/www/vhosts/chatbot.onmascout.de/chatbot && /var/www/vhosts/chatbot.onmascout.de/chatbot/.venv/bin/python manage.py purge_old_data >> /var/log/purge_old_data.log 2>&1
```

| Job | Schedule | Purpose |
|---|---|---|
| Database backup | Daily, 2:00 AM IST | Dumps `onma_chatbot` to `/root/backups/`, keeps last 7 backups, deletes older ones |
| Data purge | Weekly, Sunday 3:00 AM IST | Runs Django's `purge_old_data` management command — deletes chat sessions/messages older than `RETENTION_DAYS` (90 days), per GDPR-style retention policy |

Backup script location: `/root/backup_onma_chatbot.sh`
```bash
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
PGPASSWORD='Onma@#56ChatbotSecure67' pg_dump -h localhost -U onma_user -F c onma_chatbot -f /root/backups/onma_chatbot_$TIMESTAMP.dump
cd /root/backups && ls -t onma_chatbot_*.dump | tail -n +8 | xargs -r rm --
```

---

## 9. Recommendations / What Should Be Done Next

### High priority
1. **Test the live chatbot end-to-end** — send a real message via `/demo/` and confirm it gets an LLM reply and saves to the database correctly, not just that pages load.
2. **Log into `/admin/`** with the migrated superuser account and spot-check that `bot_knowledgeentry`, `bot_lead`, and other data look correct after migration.
3. **Verify off-server backup copies** — current backups live only on the same VPS (`/root/backups/`). If the server itself fails, backups are lost too. Consider syncing backups to a separate location (e.g., rclone to Google Drive/S3, or periodic download to a local machine).
4. **Buy a real Plesk license, or note the 15-day trial expiry** — Plesk is currently on a trial license. Check 1blu's VPS plan for whether a Plesk license is bundled; if not, decide whether to purchase one or switch to a non-Plesk workflow before it expires.

### Medium priority
5. **Lock down CORS** — `CORS_ALLOW_ALL_ORIGINS=True` is currently set (inherited from Render/dev defaults). Per the code comment in `settings.py` itself, this should be set to `False` with the real site origin(s) explicitly listed in `CORS_ALLOWED_ORIGINS` before this is considered fully production-hardened.
6. **Run Django's deployment checklist**:
   ```bash
   python manage.py check --deploy
   ```
   This surfaces any remaining security warnings beyond what's been manually addressed here.
7. **Set up monitoring/alerting** — currently no alerting if the site goes down, the backup cron fails silently, or disk space runs low on the VPS. Consider a simple uptime monitor (e.g., UptimeRobot, free tier) pointed at `https://chatbot.onmascout.de/demo/`.
8. **Rotate the OpenAI API key** — since it has been pasted in multiple places during this migration (chat, files), consider rotating it in the OpenAI dashboard once everything is confirmed stable, as a general security hygiene practice.

### Lower priority / housekeeping
9. **Delete the old `purge_cron.bat`** from the repo — it's a Windows-only leftover, no longer relevant now that cron handles this on Linux.
10. **Decommission Render** only after a few days of confirmed stable operation on the new VPS — keep it briefly as a fallback rather than deleting immediately.
11. **Document the Plesk Git deploy workflow** for future updates: pushing new commits to GitHub requires manually clicking "Pull"/"Deploy" in Plesk's Git tab — this is not automatic (no webhook configured). Consider setting up a GitHub webhook to Plesk for auto-deploy on push, if frequent updates are expected.
12. **Review system update cadence** — `apt upgrade` was run once during setup; set a reminder (or unattended-upgrades) to keep the VPS patched over time, since this is now a self-managed server rather than a managed platform like Render.

---

## 10. Quick Reference — Key Paths & Commands

```bash
# Project root
/var/www/vhosts/chatbot.onmascout.de/chatbot/

# Activate virtualenv
cd /var/www/vhosts/chatbot.onmascout.de/chatbot
source .venv/bin/activate

# Run Django management commands
python manage.py <command>

# Restart the app after code/env changes
# (via Plesk UI: Python tab → Restart App)
# or via SSH:
touch /var/www/vhosts/chatbot.onmascout.de/chatbot/tmp/restart.txt

# Manual database backup
/root/backup_onma_chatbot.sh

# Manual database restore (from a .dump file)
pg_restore --no-owner --no-acl -h localhost -U onma_user -d onma_chatbot <file>.dump

# View cron jobs
crontab -l

# Connect to the database directly
PGPASSWORD='Onma@#56ChatbotSecure67' psql -h localhost -U onma_user -d onma_chatbot
```