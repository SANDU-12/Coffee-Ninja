# Coffee Ninga Inquiry Desk

A staff website for logging customer questions and complaints, tracking their status, and keeping the record in PostgreSQL so nothing is lost after a refresh or a new login.

This is a two-app project:

- `frontend/` — React (Vite)
- `backend/` — Node.js + Express
- PostgreSQL via Docker

## Run it

You need Node.js 18+. PostgreSQL is started automatically by the API (embedded Postgres). Docker Desktop is optional if you prefer your own database.

```bash
cd backend
npm install
npm run db:init
npm run dev
```

To use Docker instead, run `docker compose up -d` and set `SKIP_EMBEDDED_PG=1` in `backend/.env`.

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open http://localhost:5173

### Test accounts

Password for every account: `Ninga123!`

| Role    | Email                   | What they can see                         |
| ------- | ----------------------- | ----------------------------------------- |
| Manager | maya@coffeeninga.com    | Every inquiry                             |
| Agent   | leo@coffeeninga.com     | Only inquiries Leo logged                 |
| Agent   | sam@coffeeninga.com     | Only inquiries Sam logged                 |

## Product decisions

**Access.** Managers see the whole inbox. Agents only see inquiries they created. The API checks this on every list, view, and status update. Hiding a page in React is not enough.

**Status changes.** An inquiry can move freely between New, In Progress, and Resolved. Staff can reopen a resolved case (for example, if the laptop comes back with the same fault). The backend only accepts those three values.

**Validation.** Customer name, contact, subject, and description are required. The form shows field errors immediately. The API still rejects incomplete bodies with HTTP 400, so a crafted request cannot skip the form.

**Duplicates.** If the same staff member logs the same customer name and subject again within 24 hours, the API returns HTTP 409. The form can send the record anyway with an explicit “Save anyway” confirmation (`force: true`).

**Loading and errors.** Lists and details show a loading state. Failed saves and network errors appear as messages on the page. Status updates send the last-known `updatedAt` timestamp so a stale tab cannot silently overwrite a newer change.

**Unauthorized access.** A request without a valid JWT is rejected (401). An agent who opens another agent’s inquiry URL is rejected (403), even if they type the address directly.

## Three situations to demonstrate

### 1. Invalid information

Go to **New inquiry**, leave fields empty, and click **Save inquiry**. The form explains what is missing. The API also rejects the same payload if you send it without the UI.

### 2. Two browser tabs

Open the same inquiry in two tabs. Change the status in tab A. Tab B still shows the old status until you refresh or try to save. If tab B then changes status, the API compares timestamps, returns HTTP 409 (`STALE`), and the page replaces its copy with the latest record instead of overwriting tab A. Real-time sync is intentionally not used.

### 3. Unauthorized access

Sign in as Leo. From the list, note that Sam’s inquiries are not shown. Open an inquiry ID that belongs to Sam (for a fresh database this is usually `/inquiries/4`). The API returns 403 and the page shows that the inquiry cannot be opened. Maya can open the same URL.
