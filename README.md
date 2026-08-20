# EventPulse — Real-Time Event Check-In System

**MIC Development Department — Recruitment Submission**

A check-in platform for club events: organizers create events, attendees register and receive a unique QR code, and organizers scan attendees at the door while a live dashboard updates in real time.

Built to survive concurrent traffic, screenshot sharing, and flaky venue Wi-Fi—with correctness proven, not just claimed.

## Requirement coverage

| Requirement | Implementation decision | Verifiable outcome |
| :--- | :--- | :--- |
| Create events | Organizer-only endpoint records name, date, capacity, and the creator. | A non-organizer receives `403 Forbidden`. |
| Register and receive a QR | Each registration creates a cryptographically random, unique `qr_token`. | A registration page can display only its owner's QR and status. |
| Check in by scan | The scanner sends the decoded token to an organizer-only endpoint. | First scan returns `201`; a later scan returns `409` with the original time and station. |
| Live dashboard | Socket.IO broadcasts only to the event room after a committed check-in. | Counts and checked-in list change without a page reload. |
| Capacity | Conditional database update reserves a seat before a registration is inserted. | 500 simultaneous attempts for capacity 50 produce exactly 50 registrations. |
| Offline scanner | IndexedDB persists queued scans and syncs with a client-generated `scan_id`. | Retries are safe and a conflicting late scan has an explicit outcome. |
| AI insights | Backend computes current statistics and lets the LLM only explain them. | Provider failure returns the same raw statistics, not an app error. |

## Why this approach

The task's four hard requirements—correctness under concurrency, anti-abuse, offline resilience, and grounded AI—drive the design. Registration, check-in, and the dashboard each satisfy the relevant requirement by design, rather than being made safe after basic CRUD is finished.

The stack prioritizes execution and correctness over novelty: a fully working, reliable submission is more valuable than a partially finished trendy one.

## Tech stack

| Layer | Choice | Why |
| :--- | :--- | :--- |
| **Backend** | FastAPI (Python) | Native async/await, built-in WebSocket support, and Pydantic validation. |
| **Database** | PostgreSQL | Provides real `SELECT ... FOR UPDATE`, `ON CONFLICT`, and serializable transactions. |
| **Real-time** | python-socketio | Room-based broadcasts (one room per event) plus reconnect/backoff. |
| **Frontend** | React + Vite | Meets the React requirement without unnecessary SSR overhead. |
| **QR generation** | `qrcode` (Python) | Generates a PNG server-side from a one-time token. |
| **QR scanning** | `html5-qrcode` (JavaScript) | Browser camera scanning—no native app required. |
| **Auth** | JWT (`python-jose`) | Server-side role verification on every protected route. |
| **AI** | Claude or OpenAI API | Called only from the backend; credentials remain in `.env`. |
| **Offline queue** | IndexedDB (via `idb`) | Persists scans across reloads and app termination. |

## System architecture

```mermaid
graph TD
    Attendee["Attendee Web App"] -->|"register / view QR"| API["FastAPI Backend"]
    Organizer["Organizer Dashboard"] -->|"create event / export / ask"| API
    ScannerA["Scanner — Station A"] -->|"scan online / queue offline"| API
    ScannerB["Scanner — Station B"] -->|"scan online"| API

    API -->|"atomic transaction"| DB[("PostgreSQL")]
    API -->|"context + question"| LLM["AI Insights Module"]
    LLM -->|"answer / fallback"| API

    API -.->|"broadcast checkin:new"| Socket["Socket.IO Server"]
    Socket -.->|"live updates"| Organizer
```

## Database schema

The two most important constraints are `registrations.qr_token` being unique and `checkins.registration_id` being unique. The concurrency and anti-abuse mechanisms rely on them.

```mermaid
erDiagram
    USERS ||--o{ EVENTS : organizes
    USERS ||--o{ REGISTRATIONS : registers
    EVENTS ||--o{ REGISTRATIONS : has
    REGISTRATIONS ||--o| CHECKINS : produces
    REGISTRATIONS ||--o{ SYNC_CONFLICTS : may_flag

    USERS {
        uuid id PK
        string role "organizer or attendee"
    }
    EVENTS {
        uuid id PK
        int capacity
        int seats_remaining
    }
    REGISTRATIONS {
        uuid id PK
        string qr_token UK "one-time token"
    }
    CHECKINS {
        uuid id PK
        uuid registration_id FK "unique"
        uuid scan_id UK "sync idempotency key"
    }
    SYNC_CONFLICTS {
        uuid id PK
        uuid registration_id FK
    }
```

## Core features

- **Event creation:** An organizer creates an event with name, date, and capacity; the service initializes `seats_remaining` to capacity.
- **Registration:** An attendee registers for an event, receives a one-time `qr_token`, and can view the generated QR PNG and their own status.
- **Scan-to-check-in:** An organizer browser decodes the token and posts it to `/checkins`; the response always states whether the scan was accepted or rejected and why.
- **Live dashboard:** A Socket.IO room per event receives `checkin:new` only after the database transaction commits, including the updated count and attendee summary.
- **Roles:** API routes use `Depends(require_role("organizer"))`, and event access additionally checks that the organizer owns the event. Attendees can access only their own registrations.
- **Export:** `/events/{id}/export.csv` streams attendee data and check-in timestamps.

### Suggested endpoint contract

| Method | Route | Access | Purpose |
| :--- | :--- | :--- | :--- |
| `POST` | `/events` | Organizer | Create an event. |
| `POST` | `/events/{event_id}/registrations` | Attendee | Reserve a seat and create a registration/QR token. |
| `GET` | `/registrations/{registration_id}/qr` | Registration owner | View the attendee's QR code. |
| `POST` | `/checkins` | Event organizer | Check in a decoded QR token. |
| `POST` | `/checkins/sync` | Event organizer | Sync an offline scan, keyed by `scan_id`. |
| `GET` | `/events/{event_id}/dashboard` | Event organizer | Fetch current attendee and check-in data. |
| `GET` | `/events/{event_id}/export.csv` | Event organizer | Export registrations and check-in timestamps. |
| `POST` | `/events/{event_id}/insights` | Event organizer | Ask a natural-language question about current event data. |

## Hard requirement 1 — Concurrency and locking

### Duplicate check-in prevention

A second scan of the same token must never create another check-in, including when requests reach separate processes. PostgreSQL enforces that rule with a unique constraint:

```sql
INSERT INTO checkins (registration_id, station_id, checked_in_at)
VALUES ($1, $2, now())
ON CONFLICT (registration_id) DO NOTHING
RETURNING *;
```

If no row is returned, the attendee has already checked in. The API performs a plain `SELECT` to return `409 Conflict` with the original check-in time and station, such as “Already checked in at 6:42 PM at Station B.” The scanner shows this outcome prominently; it is never silently ignored.

### Capacity enforcement

`seats_remaining` is decremented and checked within one atomic transaction:

```sql
BEGIN;

UPDATE events
SET seats_remaining = seats_remaining - 1
WHERE id = $1 AND seats_remaining > 0
RETURNING seats_remaining;

-- If no row is returned: ROLLBACK and respond 409 "Event full".
INSERT INTO registrations (event_id, attendee_id, qr_token)
VALUES ($1, $2, $3);

COMMIT;
```

### Multi-process proof

Run two API processes against the same PostgreSQL database, then point the test script at both ports:

```bash
# Terminal 1
uvicorn app.main:app --port 8000

# Terminal 2 — same DATABASE_URL
uvicorn app.main:app --port 8001

# Terminal 3
python scripts/load_test.py --base-urls http://localhost:8000 http://localhost:8001
```

`scripts/load_test.py` sends 500 concurrent registration requests to an event with capacity 50, then at least 100 concurrent scans of a single QR. It must print and save the following assertions (for example at `docs/load-test-output.txt` or as a screenshot committed with the submission):

```text
capacity test: 50 created, 450 rejected, 50 registrations in database  PASS
duplicate scan test: 1 accepted, 99 rejected as already checked in, 1 check-in in database  PASS
```

The exact number of duplicate attempts may be higher, but the invariant is fixed: exactly 50 registrations and exactly one check-in. This proof exercises separate server processes; no in-memory lock is part of the correctness guarantee.

## Hard requirement 2 — QR sharing and screenshot abuse

QR codes contain opaque, cryptographically random tokens (`secrets.token_urlsafe`, stored under a unique database constraint). Each token is usable once: the successful check-in row makes it invalid server-side immediately. Any later scan—whether from the original phone or a screenshot—returns `409 Already checked in` with the original check-in time.

Short-lived rotating tokens would better resist screenshots shared before the event, but require an attendee to be online at the door. Because venue connectivity is unreliable, this single-entry design prioritizes offline attendee capability. Preventing all pre-entry sharing ultimately requires identity verification.

## Hard requirement 3 — Offline-first scanning

The scanner is a PWA. Each scan is written to an IndexedDB queue immediately, before any network request, and the UI presents an optimistic “Scanned — will confirm” state. A background loop retries queued scans when connectivity returns. Every queued item has a client-generated UUID `scan_id`; the server persists it under a unique constraint so retrying the same sync request returns its prior result instead of duplicating a check-in.

### Offline conflict policy

Consider this sequence:

1. Station A scans an attendee offline at 6:40.
2. Station B scans the same attendee online at 6:41.
3. Station A reconnects and syncs at 6:45.

Server-arrival order is authoritative. Station A reaches the `ON CONFLICT DO NOTHING` path, receives no inserted row, and a conflict is logged in `sync_conflicts`. The synced item is retained locally with a visible conflict state, and the response says: “Already checked in at Station B, 6:41 PM.” Client timestamps are not trusted because device clocks can drift or be manipulated; logging the conflict avoids rewriting check-in history after admission.

## Hard requirement 4 — AI-powered event insights

The AI is a narrator, not a calculator. For each request, the backend queries PostgreSQL and calculates `checked_in`, `registered`, `no_shows`, `no_show_percentage`, `spots_remaining`, and `peak_checkin_window`. It supplies those values as the only source of truth in the model context and instructs the model not to invent statistics.

The dashboard supports questions including:

- “How many people have checked in so far?”
- “What percentage of registered attendees are no-shows?”
- “What time did check-ins peak?”
- “How many spots are left?”

The AI request has an explicit five-second timeout. On timeout or provider error, the endpoint returns:

```json
{
  "ai_available": false,
  "stats": {
    "checked_in": 0,
    "spots_remaining": 0,
    "peak_checkin_window": null
  }
}
```

The frontend shows a loading indicator while the insight request is pending, then renders the raw statistics as a plain card if `ai_available` is `false`. An unavailable AI provider degrades the insight feature, never the event app.

## Folder structure

```text
eventpulse/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── models.py
│   │   ├── routes/
│   │   ├── sockets.py
│   │   ├── auth.py
│   │   └── ai.py
│   ├── scripts/
│   │   └── load_test.py
│   ├── requirements.txt
│   └── .env.example             # Defines required LLM_API_KEY, DATABASE_URL
├── docs/
│   └── load-test-output.txt      # Captured concurrency proof for submission
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   └── lib/
│   │       ├── offlineQueue.js
│   │       └── socket.js
│   └── package.json
└── README.md
```

## Setup and local development

### 1. Backend

```bash
cd backend
python -m venv venv

# macOS/Linux
source venv/bin/activate

# Windows PowerShell
# .\\venv\\Scripts\\Activate.ps1

pip install -r requirements.txt

# Copy the environment template and add database/LLM credentials.
cp .env.example .env

# Server process 1 (terminal 1)
uvicorn app.main:app --reload --port 8000

# Server process 2 (terminal 2), for multi-process concurrency verification.
# Both processes must use the exact same DATABASE_URL.
uvicorn app.main:app --reload --port 8001
```

Configure backend CORS to allow the Vite development origin (normally `http://localhost:5173`). The two API ports are for the concurrency test; the load-test script calls both directly and does not depend on CORS.

### 2. Frontend

```bash
cd frontend
npm install
npm run dev
```

## Environment variables

Start from `backend/.env.example` and provide at least:

```dotenv
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/eventpulse
LLM_API_KEY=your_provider_key
FRONTEND_ORIGINS=http://localhost:5173
```

Keep `.env` out of version control. The frontend must never receive the LLM API key.
