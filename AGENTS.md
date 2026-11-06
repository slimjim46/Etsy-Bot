# AGENTS.md — Etsy → Instagram Auto-Poster

Instructions for AI coding agents (OpenCode) working in this repo. Read this first, then `docs/PLAN.md` and `docs/TASKS.md`.

## Project in one paragraph
A small always-on Java 21 service that polls an Etsy shop for new active listings, builds a caption, and publishes each listing to the shop owner's Instagram (Business/Creator account) via the Instagram Graph API. It supports an approval mode (queue first) and full auto mode. State lives in SQLite. The shop owner never installs or runs anything.

## Stack (do not change without asking)
- Java 21, Maven (single module), package root `shop.poster`
- `java.net.http.HttpClient` for all HTTP (no OkHttp/Apache/Retrofit)
- Jackson (`jackson-databind`) for JSON
- Javalin 6 for the tiny web server (OAuth callbacks, approval page, health check)
- SQLite via `org.xerial:sqlite-jdbc`, plain JDBC (no ORM)
- SLF4J + `slf4j-simple` for logging
- JUnit 5 + Mockito for tests; use a local `com.sun.net.httpserver` or WireMock for HTTP fakes
- Scheduling: `ScheduledExecutorService`. No Spring.

## Commands
- Build + test: `mvn -q verify`
- Run: `mvn -q exec:java -Dexec.mainClass=shop.poster.Main` (or `java -jar target/poster.jar`)
- Single test: `mvn -q -Dtest=ClassName test`
- Always run `mvn -q verify` before declaring a task done.

## Layout
```
src/main/java/shop/poster/
  Main.java  Config.java  Poller.java
  etsy/        EtsyAuth, EtsyClient, EtsyListing (record)
  instagram/   InstagramAuth, InstagramClient, ImagePrep
  caption/     CaptionBuilder, AiCaptionWriter (optional)
  store/       Store (SQLite), Schema
  alert/       Alerter
  web/         WebServer (Javalin routes)
src/test/java/shop/poster/...
docs/          PLAN, ARCHITECTURE, SETUP, TASKS, DEPLOYMENT, CONFIG, PRIVACY_POLICY
```

## Hard rules
1. **Never hardcode secrets.** All keys/tokens come from env vars (see `docs/CONFIG.md`) or the `tokens` table. Never log tokens, auth codes, or API keys. Redact in logs.
2. **Mark a listing `posted` only after Instagram `media_publish` returns an id.** Never before.
3. **Seed on first run**: if the `seeded` flag is unset, record every current listing as `seen` and post nothing. Setting the flag happens after seeding succeeds.
4. **Only `state=active` listings** are eligible.
5. **Etsy refresh tokens rotate.** After every refresh, persist the new refresh token in the same transaction/step before using the new access token.
6. **Instagram long-lived token** must be refreshed before expiry (target: refresh when < 20 days left, check daily).
7. **Failures alert** via `Alerter` (token refresh failure, publish failure after retries, seed failure). Retry cap per listing = 3 attempts, then status `failed` and alert once.
8. **Idempotency**: a listing id can never be published twice. Use the DB status as the source of truth and a unique primary key on `listing_id`.
9. **No prohibited automation**: no auto-follow, auto-like, mass commenting, scraping, or unofficial/private APIs. Only official Etsy Open API v3 and Meta Graph API.
10. **Don't guess API details.** Etsy v3 and Meta Graph API change. If an endpoint, scope, field, or limit is uncertain, say so and check current official docs (developers.etsy.com, developers.facebook.com) before coding against it. Isolate API specifics in the client classes so changes are local.

## Code style
- Java records for DTOs; immutable where practical; no Lombok.
- Constructor injection by hand in `Main`; small classes, one responsibility.
- Clients take a `HttpClient` and base URL in the constructor so tests can point at a fake server.
- Checked errors become a small set of unchecked exceptions: `ApiException` (with status + redacted body), `AuthException`.
- Timeouts on every HTTP request (connect 10s, request 30s). Retry only idempotent GETs and token refresh, with backoff; never auto-retry `media_publish` blindly (check state first).
- Keep methods short; comment *why*, not *what*.

## Testing expectations
- Unit tests for: `CaptionBuilder`, `ImagePrep` (aspect-ratio math), `Store` (against temp SQLite file), `Poller` decision logic (seed, dedupe, approval vs auto, retry cap), token-expiry logic.
- Client tests against a fake HTTP server with recorded-style JSON fixtures in `src/test/resources/fixtures/`.
- No test may hit the real Etsy or Meta APIs.

## Workflow for agents
- Work through `docs/TASKS.md` in order. One milestone at a time; tick the checkbox when its acceptance criteria pass.
- Before large changes, state a short plan. Prefer small commits-worth of change.
- If a requirement conflicts with these rules or the docs, stop and ask rather than improvising.
- Update the relevant doc when behavior or config changes.
