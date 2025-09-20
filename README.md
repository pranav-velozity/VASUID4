# UID Ops Backend

Express + SQLite backend for the UID Ops UI.  
Features:
- Records upsert via `PATCH /records/:id` (triggers SSE on first completion)
- Query & export to `.xlsx`
- **Weekly plan persistence** team-wide via `GET/PUT /plan/weeks/:mondayISO`
- SSE stream for live “Cardio” (`/events/scan`)

## Quick start (local)

```bash
npm install
npm start
# server on http://localhost:4000
```

Frontend: set `VITE_API_BASE=http://localhost:4000` (or use the `<meta name="api-base">` in your single-file UI).

## Env vars

- `PORT` (default `4000`)
- `ALLOWED_ORIGIN` (default `*`; set to your Netlify domain in prod)
- `DB_DIR` (default `./data`)
- `DB_FILE` (default `${DB_DIR}/uid_ops.sqlite`)

## API

### Records
- `PATCH /records/:id`  
  Body: `{ "field": "sku_code", "value": "SKU123" }`  
  Creates the record if missing; when all required fields are set (`po_number`,`sku_code`,`uid`,`mobile_bin`), status flips to `complete` and emits SSE.

- `GET /records?from=YYYY-MM-DD&to=YYYY-MM-DD&status=complete&limit=50000`

- `GET /export/xlsx?date=YYYY-MM-DD` → downloads Excel

### Weekly Plan (team-wide persistence)
- `PUT /plan/weeks/:mondayISO`  
  Body: array of plan rows:
  ```json
  [
    {
      "po_number": "PO123",
      "sku_code": "SKU9",
      "start_date": "2025-09-15",
      "due_date": "2025-09-19",
      "target_qty": 120,
      "priority": "High",
      "notes": "red labels"
    }
  ]
  ```
- `GET /plan/weeks/:mondayISO` → returns array (or `[]`)
- `GET /plan/weeks` → recent weeks with plans

### SSE (Cardio)
- `GET /events/scan` (Server-Sent Events)
  - Emits `{ ts: ISOString }` when a record first becomes `complete`.
  - Keep-alive is handled by the client.

## Deploy on Render

1. Push this repo to GitHub.
2. **Create Web Service** → pick this repo.
3. Set env vars:
   - `ALLOWED_ORIGIN` → `https://<your-netlify-site>.netlify.app`
   - `DB_DIR`        → `/var/data`
4. Add a **persistent disk**:
   - Mount path `/var/data`, size 1 GB.
5. Build Command: `npm install`  
   Start Command: `node server.js`

The service will expose URLs like:
- `https://your-render-service.onrender.com/records`
- `https://your-render-service.onrender.com/plan/weeks/2025-09-15`
- `https://your-render-service.onrender.com/events/scan`

## Notes

- SQLite is great for MVP + small teams. If you need more throughput, you can swap to Postgres with minimal changes (same endpoints).
- The SSE endpoint adds `Access-Control-Allow-Origin` explicitly so your Netlify UI can subscribe cross-origin.
