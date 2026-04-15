# Gotchas

Traps, quirks, and things not to change without care.

---

## Never edit `sqlc/db/` or `docs/swagger.*` directly

These files are auto-generated. Manual edits are overwritten on the next `make sqlc` or `make swagger`. Always edit the source (`sqlc/query.sql` or controller/DTO annotations) and regenerate.

---

## Update mocks after any interface change

After modifying `IRepository`, `IUrls`, `IAuth`, or `IRedirects`, run `mockery --all` to regenerate the mocks in `mocks/`. Stale mocks cause test compilation failures.

---

## Basic Auth password is empty in `.env.production`

The production `BASIC_AUTH_PASSWORD` is injected at build time via the GitHub Actions secret `BASIC_AUTH_PASSWORD`. If you run a production build locally without this secret, Basic-Auth-protected endpoints will accept any password. Always use `.env.development` for local testing.

---

## Short URL uniqueness relies on a partial index

The uniqueness constraint on `short_url` is a **partial index**: `CREATE UNIQUE INDEX ON urls (short_url) WHERE is_deleted = false`. A soft-deleted URL's short code can be reused. SQLC-generated insert functions rely on this behavior. Do not convert soft deletes to hard deletes — it would break the partial index semantics.

---

## Deferred error setting pattern in controllers

Controllers use:
```go
defer func() { reqctx.GetValue(ctx).SetError(err) }()
```
This means the error is reported to the request context (for logging) at the end of the handler, regardless of which code path set `err`. Do not refactor this to an inline call — the deferred form captures the final value of `err`, including values set after the defer statement.

---

## Rate limiting config is loaded at startup

`ratelimit.yaml` is read once when the server starts. Changes require a server restart. Rate limit values are keyed by endpoint path — if you add a new route in `endpoints/endpoints.go`, add a corresponding entry in `ratelimit.yaml` or it will use default (permissive) limits.

---

## Swagger annotations must follow `swag` syntax exactly

The `swag` tool is strict about annotation format. Missing a required field (e.g., `@Success` response type) or using incorrect types causes `make swagger` to fail or generate incomplete docs. Refer to existing annotations in `controller/` for the correct format.

---

## Redirect tracking endpoint uses Basic Auth, not Bearer

`POST /api/urls/v1/redirect/:shortLink` is called by the Zap frontend for unauthenticated visitors (who have no JWT). It uses `authApi` (Basic Auth) in the frontend. Do not change this to Bearer-only authentication — it would break redirect tracking for non-logged-in users.

---

## `APP_URLSHORTENER_` prefix for all environment variables

All config fields are loaded via `envconfig.MustProcess("APP_URLSHORTENER", &cfg)`. The prefix `APP_URLSHORTENER_` is automatically prepended. If a variable is named `DbPassword` in the struct, the env var is `APP_URLSHORTENER_DBPASSWORD`. Lowercase matching is also supported.
