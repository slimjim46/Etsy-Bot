# Configuration Reference

All config is environment variables, loaded once by `Config.java`, validated at startup (fail fast with a clear message listing what's missing). See `.env.example`.

| Var | Required | Default | Notes |
|---|---|---|---|
| POSTING_MODE | no | approval | `approval` or `auto` |
| POLL_INTERVAL_MINUTES | no | 20 | clamp 5–60 |
| DRY_RUN | no | false | never calls publish; logs would-be posts |
| PUBLIC_BASE_URL | yes | | HTTPS; used to build redirect URIs |
| PORT | no | 8080 | |
| ADMIN_PASSWORD | yes | | HTTP Basic for admin routes |
| ETSY_API_KEY | yes | | |
| ETSY_SHARED_SECRET | maybe | | per current Etsy docs |
| ETSY_SHOP_ID | no | auto | discovered after auth |
| META_APP_ID / META_APP_SECRET | yes | | |
| IG_USER_ID | no | from DB | stored after connect |
| CAPTION_HASHTAGS_FIXED | no | | space-separated |
| CAPTION_MAX_HASHTAGS | no | 15 | hard cap 30 |
| CAPTION_CTA | no | Link in bio to shop | |
| AI_CAPTIONS | no | false | needs `ANTHROPIC_API_KEY`, `AI_MODEL` |
| AI_VOICE_NOTES | no | | style guidance for AI |
| ALERT_EMAIL_TO + SMTP_* | one channel required | | |
| TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID | one channel required | | |
| DB_PATH | no | ./poster.db | point at persistent volume in prod |

## Caption template (default)
```
{title}

{description_trimmed_to_~300_chars}

Price: {currency_symbol}{price}
{CAPTION_CTA}

{fixed_hashtags} {listing_tag_hashtags}
```
Rules: max 2,200 chars total; hashtags capped; strip HTML/markdown; tags → lowercase alphanumeric hashtags; dedupe; trim at word boundary with "…".
