# Architecture

## Components
```
            +-----------------+
 timer ---> |     Poller      | --- fetch active listings ---> EtsyClient --> Etsy API v3
            |                 |
            |  decide:        | --- build caption -----------> CaptionBuilder (+AiCaptionWriter)
            |  seed/dedupe/   |
            |  approval/auto  | --- prep images -------------> ImagePrep
            |                 |
            |                 | --- publish -----------------> InstagramClient --> Graph API
            +--------+--------+
                     |
                  Store (SQLite)         Alerter (email / Telegram)
                     |
            +--------+--------+
            |   WebServer     |  /connect/etsy, /callback/etsy,
            |   (Javalin)     |  /connect/instagram, /callback/instagram,
            +-----------------+  /approvals, /health
```

## Poll cycle
```
tick():
  ensureFreshTokens()                       # Etsy + Instagram
  listings = etsy.getActiveListings()
  if !store.isSeeded():
      store.markAllSeen(listings); store.setSeeded(); return
  for l in listings where store.status(l.id) is absent:
      caption = captionBuilder.build(l)
      if mode == approval: store.queue(l.id, caption)
      else: publish(l, caption)
  for q in store.approvedAndRetryable():
      publish(q.listing, q.caption)

publish(l, caption):
  attempts++ ; if attempts > 3 -> status=failed, alert once, return
  images = imagePrep.prepare(l.images)
  try:
      mediaId = ig.publish(images, caption)    # container(s) -> poll status FINISHED -> media_publish
      store.markPosted(l.id, mediaId)
  catch e:
      store.recordError(l.id, e); alert
```

## Listing status machine
`(none) → seen` (seeded) · `(none) → queued → approved → posted` · `queued → rejected` · `(any pending) → failed` after 3 attempts · `skipped` (inactive/ineligible).

## SQLite schema
```sql
CREATE TABLE listings (
  listing_id   INTEGER PRIMARY KEY,
  status       TEXT NOT NULL,          -- seen|queued|approved|rejected|posted|failed|skipped
  title        TEXT,
  caption      TEXT,
  image_urls   TEXT,                   -- JSON array
  attempts     INTEGER NOT NULL DEFAULT 0,
  ig_media_id  TEXT,
  last_error   TEXT,
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
);
CREATE TABLE tokens (
  provider      TEXT PRIMARY KEY,      -- etsy|instagram
  access_token  TEXT NOT NULL,
  refresh_token TEXT,
  expires_at    TEXT,                  -- ISO-8601 UTC
  extra         TEXT,                  -- JSON (e.g. ig_user_id, etsy shop_id)
  updated_at    TEXT NOT NULL
);
CREATE TABLE kv (key TEXT PRIMARY KEY, value TEXT NOT NULL);  -- seeded flag, etc.
```
Enable WAL mode. Single writer; use short transactions.

## Instagram publish flow (expected; verify in docs)
1. Single image: `POST /{ig-user-id}/media` with `image_url`, `caption` → container id.
2. Carousel: create one container per image with `is_carousel_item=true`; then `POST /{ig-user-id}/media` with `media_type=CAROUSEL`, `children=id1,id2,...`, `caption`.
3. Poll container `status_code` until `FINISHED` (timeout ~2 min; `ERROR` → fail).
4. `POST /{ig-user-id}/media_publish` with `creation_id`.
5. Never blindly retry step 4; check whether the post already exists first.

## Image handling
Etsy image URLs are public, but Instagram needs acceptable format and aspect ratio. If an image violates limits, `ImagePrep` downloads, center-crops/pads to an allowed ratio, re-encodes as JPEG, and serves it from a public URL (small `/media/{id}.jpg` route on the Javalin server, or object storage). Prefer passing the original URL untouched when it already complies.

## Auth flows
- **Etsy**: OAuth 2.0 authorization code + PKCE. `/connect/etsy` creates verifier/challenge + state (stored server-side), redirects to Etsy. `/callback/etsy` validates state, exchanges code, stores tokens. Refresh via token endpoint; refresh tokens rotate, so persist the new one immediately.
- **Instagram**: Meta OAuth → short-lived token → exchange for long-lived (~60 days) → refresh before expiry. Store IG user id.

## Security
- `/connect/*` and `/approvals` behind HTTP Basic using `ADMIN_PASSWORD`; callbacks validate `state`.
- DB file permissions 600; volume not publicly served.
- Never log secrets; redact URLs containing tokens.
- HTTPS required (platform-provided TLS or reverse proxy).
