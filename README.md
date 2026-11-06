# Etsy → Instagram Auto-Poster

Automatically posts each new active Etsy listing to an Instagram Business/Creator account. Java 21, Maven, SQLite, Javalin.

- Plan and decisions: `docs/PLAN.md`
- How it fits together: `docs/ARCHITECTURE.md`
- Account/app registration and the owner's one-time steps: `docs/SETUP.md`
- Build order for the coding agent: `docs/TASKS.md`
- Config reference: `docs/CONFIG.md` and `.env.example`
- Hosting: `docs/DEPLOYMENT.md`
- Privacy policy template (required by Meta): `docs/PRIVACY_POLICY.md`
- Agent rules: `AGENTS.md`

Quick start (after setup): copy `.env.example` to `.env`, fill it in, `mvn -q verify`, then run the jar. Visit `/connect/etsy` and `/connect/instagram` once to authorize.
