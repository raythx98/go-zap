# Integrations

External services — contracts, auth, and failure modes.

---

## PostgreSQL

**Used for**: all persistent data — users, URLs, redirects.

**Auth**: username/password via `APP_URLSHORTENER_DBUSERNAME` and `APP_URLSHORTENER_DBPASSWORD`. Connection via `pgx/v5` pool.

**Schema**: managed by golang-migrate. Tables: `users`, `urls`, `redirects`.

**Failure modes**:
- Connection failure at startup → fatal log, server does not start.
- Query error → operation returns an error; HTTP 500 or domain-appropriate status returned.
- No automatic query retry.

**Config vars**: `APP_URLSHORTENER_DBHOST`, `APP_URLSHORTENER_DBPORT`, `APP_URLSHORTENER_DBUSERNAME`, `APP_URLSHORTENER_DBPASSWORD`, `APP_URLSHORTENER_DBDEFAULTNAME`.

---

## Zap Frontend (SPA)

**Used as**: the sole consumer of this REST API.

**Auth**: JWT Bearer for authenticated endpoints; Basic Auth (`VITE_BASIC_AUTH_USERNAME`/`PASSWORD`) for public-facing endpoints.

**Contract**: Swagger spec at `/swagger/` documents all endpoints. The frontend's `src/api/` files must stay in sync with the actual API surface.

**Failure modes**:
- Token expiry → frontend interceptor auto-refreshes; if refresh fails, redirects to `/auth`.
- Network partition → frontend shows toast error.

---

## Docker Compose Stack

**Components**:

| Service | Purpose |
|---------|---------|
| `db` | PostgreSQL 17 database |
| `migrate` | Applies golang-migrate migrations on startup |
| `app` | The go-zap server |
| `caddy` | Reverse proxy for HTTPS/TLS termination |

**Startup order**: `db` → `migrate` → `app` → `caddy`.

**Failure modes**:
- `db` fails → migrations and app never start.
- Migration failure → app starts but encounters schema errors at runtime.
- Caddy failure → app reachable directly on port 8080 via HTTP.

---

## golang-migrate

**Used for**: database schema evolution.

**Contract**: two-file convention — `NNNNNN_name.up.sql` and `NNNNNN_name.down.sql`.

**New migration**: `make create_migration name=<descriptive_name>`.

**Failure modes**:
- Migration fails mid-run → dirty state; manual `migrate force <version>` required.

---

## Caddy (Reverse Proxy)

**Used for**: TLS termination (Let's Encrypt) and HTTP→HTTPS redirect on OCI.

**Auth**: none — traffic forwarded to port 8080.

**Failure modes**:
- Certificate issuance failure → HTTPS unavailable.
- Auto-recovers on restart.

---

## ipapi.co (External Geolocation — frontend-side)

**Note**: Geolocation of redirect events (country, city) is resolved client-side by the Zap frontend using `ipapi.co` and passed in the redirect tracking request body. The go-zap backend stores the values as provided — it does not perform its own IP geolocation.

**Failure modes**:
- `ipapi.co` rate limit or downtime → country/city fields will be empty for affected redirects.
