# Plan: Etsy → Instagram Auto-Poster

## 1. Goal
Automatically post each new Etsy listing from mom's shop to her Instagram, and help manage the page. After one-time setup she installs and runs nothing.

## 2. Decisions (locked)
| Topic | Decision |
|---|---|
| Language | Java 21 (all-Java; no TS/JS needed) |
| Build | Maven, single module |
| Web framework | Javalin (OAuth callbacks, approval page, health) |
| HTTP | `java.net.http.HttpClient` |
| JSON | Jackson |
| State | SQLite via JDBC on a persistent volume |
| Scheduler | `ScheduledExecutorService` (in-process) |
| Hosting | Always-on container or small VPS with a persistent disk (Fly.io / Railway / Render / $4–6 VPS). Not Lambda; not her computer. |
| Posting mode | Start in **approval** mode; flip to **auto** by config |
| Captions | Deterministic template first; optional AI rewrite behind a flag with template fallback |
| Website | None. Only OAuth redirect endpoints + a static privacy policy page (GitHub Pages or similar) |

## 3. How it works
1. **Etsy**: Open API v3, OAuth 2.0 with PKCE, owner authorizes once. No webhooks exist for new listings, so poll every 10–30 min (default 20) via "get listings by shop" with `state=active`, then diff against stored IDs.
2. **Caption**: title + trimmed description + price + hashtags + "link in bio" (links aren't clickable in captions). Optional AI rewrite in mom's voice.
3. **Instagram**: Instagram Graph API; account must be Business or Creator. Publishing is two steps: create media container (carousel container for multiple photos), then `media_publish`. Image URLs must be public (Etsy image URLs are). Development mode with mom added as tester avoids full app review for personal use (verify current rules).
4. **State**: SQLite holds listing statuses and tokens.
5. **Alerts**: email and/or Telegram on any failure.

## 4. Safeguards
- Seed on first run (record all existing listings as seen; post nothing).
- Mark posted only after Instagram confirms.
- Only active listings.
- Persist rotated Etsy refresh token every time.
- Refresh Instagram long-lived token (~60-day life) before expiry.
- Image prep: respect Instagram size/aspect-ratio/format limits; crop/convert if needed.
- Retry cap (3) per listing, then `failed` + one alert.
- Idempotent: one listing id → at most one post.
- Dry-run mode for testing.
- No follow/like/mass-comment automation, no scraping.

## 5. "Managing the page" scope
- **Phase 1 (core):** publish new listings.
- **Phase 2 (nice to have):** read comments, reply, hide comments; pull insights (reach, engagement, followers) into a weekly summary message.
- **Out of scope / forbidden:** follow/unfollow, auto-like, mass comments, scraping. DMs only within Meta's rules and only if explicitly added later.

## 6. Risks to verify first (before heavy coding)
1. **Etsy API access level**: register the app and confirm what approval, scopes (`listings_r`, `shops_r` expected), and rate limits a new/personal app gets.
2. **Meta Development-mode publishing**: confirm a tester account can publish via the API without app review under current rules, and which login flow applies (Instagram Login vs Facebook Login), since endpoints and token handling differ.
3. **Instagram media rules**: current image format (JPEG expected), aspect ratio window (roughly 4:5 to 1.91:1 expected), carousel size (2–10 expected), caption length (2,200 chars) and hashtag cap (30), daily publish limit. Confirm in docs.
4. **Hosting/persistent disk pricing** on the chosen provider.

Treat every API detail in these docs as "expected, verify". Check developers.etsy.com and developers.facebook.com.

## 7. Open choices remaining
- Which host (default recommendation: Fly.io with a 1 GB volume, or a small VPS).
- Alert channel: email vs Telegram (Telegram is simpler: one bot token + chat id).
- AI captions on/off at launch (default off).

## 8. Milestones
See `TASKS.md`. Order: scaffold → store → Etsy auth/client → caption → Instagram auth/client → poller → approval UI → alerts → token maintenance → deploy → go-live checklist → phase 2.
