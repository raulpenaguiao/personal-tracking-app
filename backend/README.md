# LifeLog — Backend

FastAPI server that powers the LifeLog Android app. Handles authentication, timestamped log entries, SQLite storage, and ntfy push notifications.

---

## Tech stack

| Component | Technology |
|---|---|
| Language | Python 3.11+ |
| Framework | FastAPI |
| Database | SQLite via SQLAlchemy (async) |
| Auth | JWT (python-jose) + bcrypt (passlib) |
| Notifications | ntfy HTTP API |
| Server | Uvicorn behind Nginx |
| Process manager | systemd |

---

## Project structure

```
backend/
├── main.py               ← app entry point, router registration
├── requirements.txt
├── .env.example          ← copy to .env and fill in
├── alembic/              ← database migrations
├── app/
│   ├── auth/
│   │   ├── router.py     ← /auth/login, /auth/pin, /auth/logout
│   │   ├── service.py    ← password/PIN verification, JWT issuance
│   │   └── models.py     ← User, Device, TokenBlocklist tables
│   ├── log/
│   │   ├── router.py     ← POST /log, GET /log
│   │   ├── service.py    ← entry writer, ntfy trigger
│   │   └── models.py     ← Entry table
│   ├── stats/
│   │   ├── router.py     ← GET /stats/{category}/{action}
│   │   └── service.py    ← aggregation queries
│   ├── core/
│   │   ├── config.py     ← settings loaded from .env
│   │   ├── database.py   ← SQLAlchemy engine and session
│   │   ├── security.py   ← JWT encode/decode, bcrypt helpers
│   │   └── middleware.py ← JWT auth middleware, rate limiter
│   └── notifications.py  ← ntfy HTTP client
└── tests/
```

---

## Environment variables

Copy `.env.example` to `.env`:

```bash
cp .env.example .env
```

| Variable | Description | Example |
|---|---|---|
| `SECRET_KEY` | HS256 signing key (≥32 random bytes) | `openssl rand -hex 32` |
| `ACCESS_TOKEN_EXPIRE_DAYS` | JWT TTL in days | `30` |
| `PRE_TOKEN_EXPIRE_MINUTES` | Pre-token TTL (password → PIN window) | `5` |
| `DATABASE_URL` | SQLAlchemy DB URL | `sqlite+aiosqlite:///./lifelog.db` |
| `NTFY_URL` | Base URL of your ntfy instance | `http://localhost:2586` |
| `NTFY_TOPIC` | ntfy topic name | `lifelog` |
| `ALLOWED_ORIGINS` | CORS origins (comma-separated) | `https://yourdomain.com` |

---

## Installation

### 1. Create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Run database migrations

```bash
alembic upgrade head
```

### 4. Start the development server

```bash
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

The API is now available at `http://127.0.0.1:8000`. Interactive docs at `/docs`.

---

## Production deployment

### systemd service

Create `/etc/systemd/system/lifelog.service`:

```ini
[Unit]
Description=LifeLog API
After=network.target

[Service]
User=lifelog
WorkingDirectory=/opt/lifelog/backend
EnvironmentFile=/opt/lifelog/backend/.env
ExecStart=/opt/lifelog/backend/.venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000 --workers 2
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now lifelog
```

### Nginx

See [`Arch.md`](../Arch.md) for the full Nginx configuration block. In short, proxy `/api/` to `http://127.0.0.1:8000` with TLS termination on port 443.

---

## API reference

### Auth

```
POST /api/v1/auth/login
Body: { "username": "...", "password": "..." }
→ 200 { "pre_token": "..." }   (valid for 5 minutes)

POST /api/v1/auth/pin
Body: { "pre_token": "...", "pin": "1234", "device_fingerprint": "...", "device_label": "Pixel 8" }
→ 200 { "access_token": "...", "token_type": "bearer" }

POST /api/v1/auth/logout
Headers: Authorization: Bearer <token>
→ 204
```

### Log entries

```
POST /api/v1/log
Headers: Authorization: Bearer <token>
Body: { "category": "health", "action": "weight", "value": "82.3", "note": "..." }
→ 201 { "id": 412, "created_at": "2025-03-16T09:14:00Z" }

GET /api/v1/log?category=health&limit=50
Headers: Authorization: Bearer <token>
→ 200 [ { "id": 412, "action": "weight", "value": "82.3", "created_at": "..." }, ... ]
```

### Stats

```
GET /api/v1/stats/health/weight?days=30
Headers: Authorization: Bearer <token>
→ 200 { "points": [ { "timestamp": "...", "value": 82.3 }, ... ] }
```

---

## Categories and actions

| Category | Example actions |
|---|---|
| `health` | weight, steps, sleep_hours, mood, workout |
| `mind` | journal, mood, meditation_minutes, gratitude |
| `finances` | expense, income, savings_balance |
| `projects` | update, milestone, time_logged |
| `learning` | book_progress, course_progress, skill_note |
| `social` | interaction, event, gratitude |

These are free-text strings in phase 1 — no validation is enforced beyond non-empty.

---

## Running tests

```bash
pytest tests/ -v
```

Tests use an in-memory SQLite database and mock the ntfy client.

---

## ntfy setup

The backend expects a self-hosted ntfy instance. The simplest setup on the same VPS:

```bash
sudo apt install ntfy
sudo systemctl enable --now ntfy
```

Default port is `2586`. Set `NTFY_URL=http://localhost:2586` in `.env`. On the Android device, install the ntfy app and subscribe to your topic.
