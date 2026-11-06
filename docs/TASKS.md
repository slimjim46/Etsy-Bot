# Tasks (build order for the coding agent)

Work top to bottom. Each milestone is done only when its acceptance criteria pass and `mvn -q verify` is green. Tick the box when finished. Verify API specifics against official docs as you reach them.

## M0 — Scaffold
- [ ] `pom.xml` (Java 21, deps per AGENTS.md, shade plugin → `target/poster.jar` with Main-Class)
- [ ] `Config.java` loads/validates env vars (fail fast, list what's missing)
- [ ] `Main.java` wires components, starts Javalin and scheduler, graceful shutdown
- [ ] `/health` route
**Accept:** app starts with a valid env, exits clearly with a missing one; `/health` returns 200.

## M1 — Store
- [ ] `Schema` creates tables from ARCHITECTURE.md; WAL on
- [ ] `Store`: seeded flag, upsert/get listing status, queue/approve/reject, markPosted, recordError, token get/put
**Accept:** tests on a temp DB cover status transitions, duplicate-id safety, token persistence.

## M2 — Etsy auth + client
- [ ] `EtsyAuth`: PKCE verifier/challenge, state storage, `/connect/etsy`, `/callback/etsy`, token exchange
- [ ] Token refresh with **rotation persisted immediately**; refresh when < 5 min left
- [ ] `EtsyClient`: discover shop id, fetch active listings with pagination, fetch images, map to `EtsyListing` record (id, title, description, price, currency, tags, imageUrls, state, url)
**Accept:** fake-server tests for auth exchange, rotation, pagination, 401→refresh→retry once. Manual: real connect works and prints listing titles in dry-run.

## M3 — Captions
- [ ] `CaptionBuilder` per CONFIG.md template, with hashtag/length rules
- [ ] `AiCaptionWriter` (optional, flagged) calling Anthropic API; falls back to template on any error/timeout; output validated (length, no links, hashtag cap)
**Accept:** table-driven tests (long descriptions, HTML, emoji, no tags, 40 tags, 2,200-char boundary).

## M4 — Instagram auth + client + images
- [ ] `InstagramAuth`: connect/callback, short→long-lived token exchange, store IG user id
- [ ] Daily token refresh job (refresh when < 20 days left); alert on failure
- [ ] `ImagePrep`: check format/ratio/size; crop/convert and serve via `/media/{id}.jpg` when needed
- [ ] `InstagramClient`: single image, carousel (children → parent), poll container status, publish; clear error mapping
**Accept:** fake-server tests for single, carousel, status ERROR, timeout; ratio math unit tests. Manual: dry-run off, one real test post to a test account.

## M5 — Poller
- [ ] Seeding, dedupe, active-only, approval vs auto, retry cap, dry-run
- [ ] Never blind-retry publish; check state first
- [ ] Single in-flight cycle (no overlapping ticks)
**Accept:** tests for: first run posts nothing; restart doesn't duplicate; failure increments attempts and stops at 3; mode switch works.

## M6 — Approval UI
- [ ] `/approvals` (Basic auth): list queued with images + caption, edit caption, Approve / Reject
- [ ] Optional: notify via Alerter channel when something is queued
**Accept:** approving moves status to `approved` and next tick publishes it.

## M7 — Alerts
- [ ] `Alerter` with Telegram and/or SMTP; rate-limit duplicate alerts
- [ ] Alert on: publish failed (final), token refresh failed, seed failed, no successful poll in 2× interval
**Accept:** unit tests with fake transport; manual test message command.

## M8 — Packaging and deploy
- [ ] Dockerfile per DEPLOYMENT.md; volume at `/data`
- [ ] Deploy to chosen host; set secrets; confirm HTTPS and both redirect URIs
- [ ] Nightly DB backup
**Accept:** container restarts keep state; `/health` reachable.

## M9 — Go-live
Run the checklist in `SETUP.md` §F. Keep approval mode 1–2 weeks, then switch to auto.

## M10 — Phase 2 (optional)
- [ ] Comments: list, reply, hide (official endpoints only; manual-trigger via admin page first)
- [ ] Insights: weekly summary (reach, engagement, followers) sent through Alerter channel
- [ ] Never add follow/like/mass-comment/scraping features.
