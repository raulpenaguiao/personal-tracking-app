# LifeLog — System Architecture

This document describes the full architecture of LifeLog: the Android client, the VPS backend, the authentication flow, the data model, and the notification pipeline.

---

## High-level overview

```
┌─────────────────────────────────┐        HTTPS/443         ┌──────────────────────────────────────┐
│         Android app             │ ───────────────────────► │              VPS                     │
│  (Kotlin + Jetpack Compose)     │ ◄─────────────────────── │  Nginx → FastAPI → SQLite            │
│                                 │    JSON responses +      │              ↓                       │
│  Keystore: JWT token            │    push via ntfy         │           ntfy                       │
└─────────────────────────────────┘                          └──────────────────────────────────────┘
```

All communication is HTTPS. The Android app never connects on plain HTTP. Nginx terminates TLS and proxies to FastAPI on `localhost:8000`.

---

## Components

### Android app

| Layer | Technology |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose |
| Navigation | Compose Navigation |
| HTTP client | Retrofit 2 + OkHttp |
| Token storage | Android Keystore via EncryptedSharedPreferences |
| Charts | Vico (or MPAndroidChart) |
| Notifications | ntfy Android app / ntfy subscription |

The app is organised around six category screens, each with a list of quick-log actions and a charts/trends tab. A single `LogRepository` handles all API calls and injects the auth token into every request via an OkHttp interceptor.

### VPS backend

| Layer | Technology |
|---|---|
| Language | Python 3.11+ |
| Framework | FastAPI |
| Database | SQLite (via SQLAlchemy) |
| Auth | JWT (python-jose) + bcrypt |
| Notifications | ntfy HTTP API |
| Reverse proxy | Nginx |
| TLS | Let's Encrypt via certbot |
| Process manager | systemd |

The backend exposes a versioned REST API at `/api/v1/`. It validates the JWT on every protected route, writes the log entry, then fires a POST to the local ntfy server.

---

## Authentication flow

```
First login on a new device
───────────────────────────
App                         Server
 │─── POST /auth/login ───►│  verify password (bcrypt)
 │◄── { pre_token } ───────│  short-lived token (5 min TTL)
 │
 │  User enters PIN
 │─── POST /auth/pin ──────►│  verify PIN, record device fingerprint
 │◄── { access_token } ────│  JWT (configurable TTL, default 30 days)
 │
 │  Token stored in Keystore


Returning recognised device
───────────────────────────
App                         Server
 │─── POST /auth/pin ──────►│  device fingerprint known, verify PIN only
 │◄── { access_token } ────│


Every protected request
───────────────────────
 │─── GET/POST /api/v1/... ─►│  Authorization: Bearer <token>
 │                           │  middleware validates JWT signature + expiry
 │◄── 200 / 401 ────────────│
```

The device fingerprint is a SHA-256 hash of the Android `ANDROID_ID` combined with a server-side salt. It is stored in the `devices` table alongside the user ID and a `last_seen` timestamp.

---

## Data model

All categories share the same generic log table in phase 1. This will be migrated to per-field schemas in a future version.

```sql
-- Users
CREATE TABLE users (
    id          INTEGER PRIMARY KEY,
    username    TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    pin_hash    TEXT NOT NULL,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Known devices
CREATE TABLE devices (
    id           INTEGER PRIMARY KEY,
    user_id      INTEGER REFERENCES users(id),
    fingerprint  TEXT NOT NULL,
    label        TEXT,               -- e.g. "Pixel 8"
    last_seen    DATETIME
);

-- Log entries (phase 1: generic)
CREATE TABLE entries (
    id          INTEGER PRIMARY KEY,
    user_id     INTEGER REFERENCES users(id),
    category    TEXT NOT NULL,       -- health | mind | finances | projects | learning | social
    action      TEXT NOT NULL,       -- e.g. "weight", "mood", "expense"
    value       TEXT,                -- numeric or free text
    note        TEXT,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Future phase: each category gets its own table with typed columns (e.g. `health_weight` with a `kg REAL` column, `finances_expense` with `amount REAL` and `currency TEXT`).

---

## API endpoints

### Auth

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/auth/login` | Password check, returns pre-token |
| POST | `/api/v1/auth/pin` | PIN check (+ device registration on first use), returns JWT |
| POST | `/api/v1/auth/logout` | Invalidates the current token (server-side blocklist) |

### Logging

| Method | Path | Description |
|---|---|---|
| POST | `/api/v1/log` | Create a new timestamped entry |
| GET | `/api/v1/log?category=health` | List entries for a category |

### Stats (for charts)

| Method | Path | Description |
|---|---|---|
| GET | `/api/v1/stats/{category}/{action}` | Aggregated time-series data for graphing |

All endpoints except the auth ones require `Authorization: Bearer <token>`.

### Example log request

```json
POST /api/v1/log
Authorization: Bearer eyJ...

{
  "category": "health",
  "action": "weight",
  "value": "82.3",
  "note": "After morning run"
}
```

### Example stats response

```json
GET /api/v1/stats/health/weight

{
  "action": "weight",
  "unit": "kg",
  "points": [
    { "timestamp": "2025-01-01T08:00:00Z", "value": 84.1 },
    { "timestamp": "2025-01-08T08:00:00Z", "value": 83.5 }
  ]
}
```

---

## Notification pipeline

Every successful write to the database triggers a fire-and-forget HTTP POST to the ntfy server:

```
FastAPI log writer
      │
      ▼
POST http://localhost:2586/lifelog
  Headers:
    Title:    Health · weight logged
    Priority: low
    Tags:     white_check_mark
  Body:
    82.3 kg — After morning run
```

The ntfy topic (`lifelog`) is subscribed to on the Android device via the ntfy app. No external ntfy relay is used — the server talks to the local ntfy instance directly.

---

## Infrastructure layout on the VPS

```
/opt/lifelog/
├── backend/
│   ├── main.py
│   ├── .env              ← secrets, never committed
│   ├── lifelog.db        ← SQLite database
│   └── ...
/etc/nginx/sites-enabled/lifelog
/etc/systemd/system/lifelog.service
/var/log/lifelog/         ← application logs
```

### Nginx configuration (outline)

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$host$request_uri;
}
```

---

## Security considerations

- Passwords are hashed with bcrypt (cost factor ≥ 12)
- PINs are hashed with bcrypt as well — never stored in plain text
- JWTs are signed with HS256; the secret key is in `.env` and is ≥ 32 random bytes
- The server maintains a blocklist table for invalidated tokens (logout support)
- Device fingerprints are stored hashed, not raw
- Rate limiting is applied to all `/auth/*` endpoints via a FastAPI middleware
- The SQLite file has `chmod 640` and is owned by the service user
- Nginx drops all non-HTTPS traffic with a 301

---

## Sequence: logging a weight entry (end to end)

```
User taps "Log weight" → enters 82.3 kg → taps Save

App
  1. Reads JWT from EncryptedSharedPreferences
  2. POST /api/v1/log  { category: "health", action: "weight", value: "82.3" }
     Authorization: Bearer <token>

Nginx
  3. Terminates TLS, proxies to FastAPI

FastAPI
  4. JWT middleware: validates signature and expiry → 401 if invalid
  5. Parses request body
  6. Writes row to entries table with CURRENT_TIMESTAMP
  7. Fires POST to ntfy (async, does not block response)
  8. Returns 201 { "id": 412, "created_at": "2025-03-16T09:14:00Z" }

App
  9. Shows success snackbar
 10. Optionally refreshes chart data

ntfy
 11. Delivers push notification to subscribed devices
```

---

## Future architecture (phase 2)

- Per-category typed tables replace the generic `entries` table
- A background sync queue on the app handles offline writes
- A cron job on the VPS generates daily summaries and sends them via ntfy
- Export endpoints (`GET /api/v1/export?format=csv`) for data portability
- Optional: a simple web dashboard served by the same FastAPI app
