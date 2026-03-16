# LifeLog

A personal life-logging Android app that connects to a self-hosted VPS backend. Track health metrics, mental notes, finances, projects, learning, and social activity — all stored on your own server, with push notifications via ntfy.

---

## Project structure

```
lifelog/
├── README.md           ← you are here
├── Arch.md             ← full system architecture
├── backend/            ← FastAPI server, database, ntfy integration
│   └── README.md
└── frontend/           ← Android app (Kotlin + Jetpack Compose)
    └── README.md
```

---

## Feature overview

- **Six life categories** — Health, Mind, Finances, Projects, Learning, Social
- **Timestamped entries** — every log is written with a UTC timestamp and stored on your VPS
- **Graphs and trends** — visualise your data over time within each category
- **Secure auth** — password + PIN two-step login; recognised devices only need the PIN
- **Push notifications** — every database write triggers a notification to your self-hosted ntfy server
- **Self-hosted** — no third-party cloud; your data lives on your own VPS

---

## Quick start

### Prerequisites

| Requirement | Version |
|---|---|
| Python | ≥ 3.11 |
| Android Studio | Hedgehog or later |
| A VPS with a public domain | Ubuntu 22.04+ recommended |
| ntfy (self-hosted) | ≥ 2.0 |
| Nginx | any recent version |
| Let's Encrypt / certbot | for TLS |

### 1. Clone the repository

```bash
git clone https://github.com/youruser/lifelog.git
cd lifelog
```

### 2. Set up the backend

See [`backend/README.md`](backend/README.md) for full instructions. In brief:

```bash
cd backend
cp .env.example .env        # fill in your values
pip install -r requirements.txt
uvicorn main:app --reload
```

### 3. Set up the Android app

See [`frontend/README.md`](frontend/README.md) for full instructions. Open `frontend/` in Android Studio, set your VPS base URL in `local.properties`, and run on a device or emulator.

---

## Security notes

- Auth tokens are JWTs signed with a secret you control, stored in the Android Keystore (not SharedPreferences)
- All traffic is HTTPS — the backend refuses plain HTTP
- The PIN step is required on every new device; recognised devices are stored server-side by device fingerprint
- The `.env` file must never be committed — it is in `.gitignore`

---

## Roadmap

- [ ] Per-field database schema per category (replacing generic timestamped notes)
- [ ] Offline queue — sync entries when connectivity is restored
- [ ] Widget for quick logging from the home screen
- [ ] Export to CSV / JSON
- [ ] Backup script for the VPS database

---

## License

MIT
