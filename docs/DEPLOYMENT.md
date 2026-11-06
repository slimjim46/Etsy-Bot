# Deployment

## Requirements
Always-on process, persistent disk for SQLite, HTTPS URL, env vars/secrets. ~256–512 MB RAM is enough.

## Dockerfile
```dockerfile
FROM maven:3-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -q -B dependency:go-offline
COPY src ./src
RUN mvn -q -B package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/poster.jar poster.jar
ENV DB_PATH=/data/poster.db
VOLUME /data
EXPOSE 8080
ENTRYPOINT ["java","-Xmx256m","-jar","poster.jar"]
```
(`pom.xml` must produce `target/poster.jar` via shade/assembly plugin with `Main-Class`.)

## Option 1: Fly.io (recommended)
`fly launch`, `fly volumes create data --size 1`, mount at `/data`, `fly secrets set KEY=...` for each env var, keep exactly 1 machine (SQLite is single-writer). Verify current pricing.

## Option 2: Small VPS
Install Docker, run with `-v /srv/poster:/data --env-file .env --restart unless-stopped`, put Caddy in front for automatic HTTPS.

## Option 3: Railway / Render
Container service with a persistent volume/disk. Check that disks are available on the chosen tier.

## Not recommended
Lambda/Cloud Functions (SQLite needs persistent disk), scheduled GitHub Actions (rotating tokens must be stored back), Cloudflare Workers (no Java), her personal computer (only works while on).

## Backups
Nightly copy of `poster.db` (`sqlite3 .backup`) to a second location. Losing it means re-authorizing and re-seeding, which is survivable but annoying.

## Operations
- `/health` returns 200 + last successful poll time.
- Logs to stdout. Alert if no successful poll in 2× interval (optional watchdog).
