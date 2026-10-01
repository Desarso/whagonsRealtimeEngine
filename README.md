# whagonsRealtimeEngine

A prototype Go service, from mid-2025, that pushes PostgreSQL row changes to browsers in real time over SockJS. It was built for a multi-tenant app whose frontend keeps a full IndexedDB copy of its data. The goal was to stream small deltas instead of refetching data, so queries in the browser could stay local and instant.

## How it works

```
Postgres (tenant DB) ── trigger → NOTIFY "whagons_tasks_changes" ──▶ pq.Listener (one per tenant)
                                                                      │
landlord DB ── tenants(name, domain, database) ──▶ tenant pool        ▼
                                                     RealtimeEngine ── SockJS /ws ──▶ authenticated browser sessions
```

- **Multi-tenant.** At startup the engine reads a `tenants` table from a "landlord" database and opens one connection pool and one `LISTEN` per tenant database (`database.go`, `publication.go`).
- **Change feed.** Database triggers send JSON payloads (`table`, `operation`, `new_data`, `old_data`, `timestamp`). The engine parses them into typed messages and broadcasts them to that tenant's sessions. The listener pings every 90 s to keep the connection alive.
- **Auth.** A client connects to `/ws` with a bearer token and its tenant `domain`. The token uses the Laravel Sanctum format (`id|plaintext`); it is SHA-256 hashed and checked against the tenant's `personal_access_tokens` table, including abilities and expiry (`auth.go`). Results are cached in memory and cleaned up every 5 minutes.
- **Session lifecycle.** Sessions start in a negotiation state and become active after authenticating. A sweep every 30 s removes zombie sessions (`websocket.go`).
- **HTTP API** (Fiber): `GET /api/health`, `GET /api/metrics`, `GET /api/sessions/count`, `POST /api/sessions/disconnect-all`, `POST /api/broadcast`.
- `test.html` is a single-page SockJS client for manual testing.

`readmes/` holds design notes for the planned sync-integrity layer: incremental row hashing, snapshot checksums, and the technology choices (wal2json, SockJS, IndexedDB).

## Run

```bash
# .env or environment
#   DB_HOST, DB_PORT, DB_USERNAME, DB_PASSWORD, DB_LANDLORD, SERVER_PORT (default 8082)
go run .
```

You need a reachable PostgreSQL landlord database with a `tenants` table, and tenant databases whose triggers `NOTIFY` on `whagons_tasks_changes`.

## Status

This is an unfinished prototype, last touched in July 2025:

- Only the tasks table is wired end to end.
- The checksum and integrity design in `readmes/` isn't implemented.
- CORS and SockJS origin checks allow everything.
- The admin endpoints (`disconnect-all`, `broadcast`) have no auth.
- `tmp/main` is a committed build artifact.

Don't deploy it as is.
