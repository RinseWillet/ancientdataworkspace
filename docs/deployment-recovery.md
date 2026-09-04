# rinsewillet.net NAS recovery runbook (Cloudflare Tunnel)

This is the **full recovery document** for rebuilding deployment from scratch using:
- Git repositories for config/code
- DockerHub images for runtime artifacts
- Cloudflare Tunnel for public ingress (no router port-forwarding)

## 1) Layman's explanation (what this setup is doing)

You have one front door to your home-hosted apps:

`Internet -> Cloudflare -> cloudflared on NAS -> nginx -> app containers`

- **Cloudflare** gives HTTPS and a stable public endpoint.
- **cloudflared** on the NAS opens an outbound-only connection to Cloudflare.
- **nginx** routes paths:
  - `/` -> landing page
  - `/webGIS/` -> AncientDataWebGIS
  - `/arcade/` -> RetroGameApp
- App containers stay on a private Docker network (`webgis-edge`).
- Router inbound port-forwards are not needed for the website path.

## 2) Manual-vs-repo gap analysis from the recent cutover

### Already codified in repos
- `RetroGameApp`:
  - router basename support in `src/App.jsx`
  - relative route paths in `src/routes.jsx`
  - compose file present with `retrogame` service
- `AncientDataWebGIS`:
  - compose env pass-through for `JWT_SECRET`, `JWT_EXPIRATION`, `DB_URL`, `DB_USER`, `DB_PASSWORD`, `CORS_ALLOWED_ORIGINS`
- `ancientdataworkspace/deploy/nginx/nginx.conf`:
  - `/arcade/` uses `proxy_pass http://retrogame:80/;` (trailing slash fix)
- `ancientdataworkspace`:
  - landing image publish workflow exists (`.github/workflows/publish-landing.yml`)

### Codified now in this cycle
- `ancientdataworkspace/deploy/docker-compose.yml`:
  - `landing` switched to image-only (removed local `build:`) for strict pull-only recovery
  - stack is proxy-only (`cloudflared`, `nginx`, `landing`) and no longer runs
    duplicate `ancientdata`/`retrogame` app services

### Operational steps that remain intentionally manual
- creating/editing NAS `.env` files with real secrets
- Cloudflare dashboard actions (tunnel hostname mapping, cache purge)
- router port-forward cleanup after validation

## 3) Domain + Cloudflare bootstrap (TransIP -> Cloudflare)

1. In Cloudflare, add `rinsewillet.net` zone (free plan).
2. At TransIP, replace nameservers with Cloudflare-assigned nameservers.
3. Wait until zone is active.
4. In Zero Trust, create Cloudflared tunnel (or reuse existing active one).
5. Add application route:
   - hostname: `rinsewillet.net`
   - path: `*` (or blank/all-path route)
   - service: `http://nginx:80`
6. SSL/TLS mode in Cloudflare zone: **Full**.

## 4) NAS prerequisites

```bash
# on NAS
docker network inspect webgis-edge >/dev/null 2>&1 || docker network create webgis-edge
```

Ensure Git and Docker Compose are available on NAS shell.

## 5) Clone/update repositories on NAS

```bash
cd /volume1/docker

git clone https://github.com/RinseWillet/AncientDataWebGIS.git || true
git clone https://github.com/RinseWillet/RetroGameApp.git || true
git clone https://github.com/RinseWillet/ancientdataworkspace.git || true

git -C /volume1/docker/AncientDataWebGIS pull --ff-only
git -C /volume1/docker/RetroGameApp pull --ff-only
git -C /volume1/docker/ancientdataworkspace pull --ff-only
```

## 6) Required NAS filesystem paths

```bash
mkdir -p /volume1/docker/ancientdata/media
mkdir -p /volume1/docker/ancientdata/backup
mkdir -p /volume1/docker/ancientdata/backups
mkdir -p /volume1/docker/ancientdata/geoserver
chown -R 1000:1000 /volume1/docker/ancientdata/media /volume1/docker/ancientdata/backup /volume1/docker/ancientdata/backups /volume1/docker/ancientdata/geoserver
find /volume1/docker/ancientdata/geoserver -type d -exec chmod 775 {} \;
find /volume1/docker/ancientdata/geoserver -type f -exec chmod 664 {} \;
```

`backup` (singular) is the in-app `NasBackupService`/`DbBackupService` bind mount; `backups` (plural) is
where the NAS cron job for `scripts/backup.sh` writes its combined DB+media+GeoServer archives — see
`AncientDataWebGIS/docs/BACKUP-STRATEGY.md`. Both are required for a). photo/media backup and c). database
backup to actually persist anywhere.

## 6a) Apply the `backup_history` schema (required — see AncientDataWebGIS/docs/architecture/adr/ADR-006-media-backup-nas-sync.md)

This app disables Flyway; all schema, including `backup_history`, must be applied manually to the shared
PostGIS database (`AncientDataWebGIS/docs/architecture/DB-MIGRATION-STRATEGY.md`). Skipping this step is the
most likely cause of the admin panel's "Back up now" button appearing broken:

```bash
psql -h 192.168.2.13 -p 2665 -U root -d webGIS_DB \
  -f /volume1/docker/AncientDataWebGIS/docs/architecture/sql/backup_history.sql
```

Verify it exists before relying on backup status reporting:

```bash
psql -h 192.168.2.13 -p 2665 -U root -d webGIS_DB -c "\d backup_history"
```

## 7) Required environment variables

### 7.1 `/volume1/docker/ancientdataworkspace/deploy/.env`

```dotenv
CLOUDFLARE_TUNNEL_TOKEN=<token>
```

### 7.2 `/volume1/docker/AncientDataWebGIS/.env`

```dotenv
DB_URL=jdbc:postgresql://192.168.2.13:2665/webGIS_DB
DB_USER=root
DB_PASSWORD=<db-password>

DATABASE_HOST=192.168.2.13
DATABASE_PORT=2665
POSTGRES_USER=root
POSTGRES_PASSWORD=<db-password>
POSTGRES_DB=webGIS_DB
GEOSERVER_ADMIN_PASSWORD=<geoserver-password>

JWT_SECRET=<long-random-secret>
JWT_EXPIRATION=86400000
CORS_ALLOWED_ORIGINS=https://rinsewillet.net
MEDIA_BASE_URL=https://rinsewillet.net/webGIS/api/media/files

# Backups (a/c requirements) — off by default; must be explicitly enabled,
# see AncientDataWebGIS/docs/BACKUP-STRATEGY.md
BACKUP_NAS_ENABLED=true
BACKUP_NAS_MOUNT_PATH=/backup/media
BACKUP_NAS_SYNC_CRON=0 0 3 * * SUN
BACKUP_DB_ENABLED=true
BACKUP_DB_OUTPUT_PATH=/backup/db
BACKUP_STALENESS_THRESHOLD_HOURS=192
```

On the NAS itself (not in this `.env` — it's read directly by the cron-invoked shell script, not the
container), also schedule `scripts/backup.sh` per `AncientDataWebGIS/docs/BACKUP-STRATEGY.md` §"Deployment
Setup", and configure Cloud Sync → Google Drive per that doc's Part 3 for off-device redundancy (b/c requirements).

## 8) Deployment order (pull-only)

```bash
cd /volume1/docker/AncientDataWebGIS
docker compose pull
docker compose up -d

cd /volume1/docker/RetroGameApp
docker compose pull
docker compose up -d --no-build

cd /volume1/docker/ancientdataworkspace/deploy
docker compose pull
docker compose up -d --no-build
```

### 8.1) Service ownership model (single source of truth)

The reverse proxy resolves `ancientdata` and `retrogame` on `webgis-edge`.
Ownership is fixed to avoid routing collisions:

- `AncientDataWebGIS` stack owns `ancientdata` and `geoserver`
- `RetroGameApp` stack owns `retrogame`
- `ancientdataworkspace/deploy` stack owns only `nginx`, `cloudflared`, `landing`

Collision check:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}' | grep -E 'ancientdata|retrogame'
```

If more than one `ancientdata` or more than one `retrogame` is running on
`webgis-edge`, nginx may route unpredictably.

## 9) Validation checks

```bash
curl -I https://rinsewillet.net/
curl -I https://rinsewillet.net/arcade/
```

Expected:
- `/` -> `200`
- `/arcade/` -> `200`

For `/webGIS/`, validate with **GET** (not `HEAD`) on API routes:

```bash
curl -sSI https://rinsewillet.net/webGIS/ | sed -n '1p;/^content-type:/Ip'
curl -sS -o /dev/null -w "%{http_code}\n" https://rinsewillet.net/webGIS/api/dashboard/summary
curl -sS -o /dev/null -w "%{http_code}\n" https://rinsewillet.net/webGIS/api/modernreferences/all
curl -sS -o /dev/null -w "%{http_code}\n" https://rinsewillet.net/webGIS/api/suggestions/my
```

Expected:
- `/webGIS/` -> `200` + `text/html`
- `/webGIS/api/dashboard/summary` -> `200`
- `/webGIS/api/modernreferences/all` -> `200` (or `204` if empty)
- `/webGIS/api/suggestions/my` -> `401` (protected endpoint)

### 9.1) Backup validation (a/b/c requirements)

```bash
# As an admin JWT, trigger both backups and confirm real outcomes (not just "ok")
curl -s -X POST https://rinsewillet.net/webGIS/api/backup/sync -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
curl -s https://rinsewillet.net/webGIS/api/backup/status -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Confirm files actually landed
ls -lh /volume1/docker/ancientdata/backup/media/ /volume1/docker/ancientdata/backup/db/

# Confirm the NAS cron archive includes all three components
tar -tzf $(ls -t /volume1/docker/ancientdata/backups/ancientdata-backup-*.tar.gz | grep -v -- '-media\.tar\.gz$\|-geoserver\.tar\.gz$' | head -1)
```

Expected `tar -tzf` output lists a `-db.sql`, `-media.tar.gz`, and `-geoserver.tar.gz` member (see
`AncientDataWebGIS/docs/BACKUP-STRATEGY.md`). If `/api/backup/sync` reports `"status":"error"`, or
`/api/backup/status` never leaves `"lastRunAt": null`, see that doc's Troubleshooting table.

`curl -I` sends `HEAD`; a `401` there does **not** necessarily mean guest `GET`
access is broken.

Internal service checks:

```bash
cd /volume1/docker/ancientdataworkspace/deploy
docker compose exec nginx wget -S -O- http://landing:80/ 2>&1 | head -n 20
docker compose exec nginx wget -S -O- http://ancientdata:8080/ 2>&1 | head -n 20
docker compose exec nginx wget -S -O- http://retrogame:80/ 2>&1 | head -n 20
docker compose exec nginx getent hosts ancientdata
docker compose exec nginx wget -S -O- http://ancientdata:8080/api/dashboard/summary 2>&1 | head -n 20
docker compose exec nginx wget -S -O- http://ancientdata:8080/api/modernreferences/all 2>&1 | head -n 20
```

## 10) Router cleanup

Only after successful validation:
- remove old Experia Box forwards for WebGIS, RetroGameApp, GeoServer, PostGISDB, PGAdmin.

## 11) Known failure modes and fixes

### A) `502` on `/webGIS/`
- backend container unhealthy/restarting
- missing required env vars (`DB_*`, `JWT_*`, `CORS_ALLOWED_ORIGINS`)
- fix backend first, then restart nginx:
```bash
cd /volume1/docker/ancientdataworkspace/deploy
docker compose restart nginx
```

Also check backend collision/missing upstream:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}' | grep ancientdata
cd /volume1/docker/ancientdataworkspace/deploy
docker compose exec nginx getent hosts ancientdata
```

### B) Arcade white screen + MIME errors for JS/CSS
- root cause was wrong proxy behavior for `/arcade/` assets
- ensure nginx has:
```nginx
location /arcade/ {
  proxy_pass http://retrogame:80/;
}
```
- then purge Cloudflare cache for `/arcade/*` (or purge all once)

### C) Cloudflare `530`
- usually tunnel hostname route mismatch/overlap in Cloudflare
- verify active tunnel has `rinsewillet.net` routed to `http://nginx:80`

### D) `/webGIS/` is `200`, but API checks show `401` with `curl -I`
- this is commonly a method mismatch (`HEAD` vs `GET`)
- re-check with:
```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://rinsewillet.net/webGIS/api/dashboard/summary
```

### E) Backend restart loop with `JwtUtil` / `JWT Secret is missing`
- active backend runtime is missing `JWT_SECRET`
- validate the `AncientDataWebGIS` runtime:
```bash
cd /volume1/docker/AncientDataWebGIS
grep -n '^JWT_SECRET=' .env
docker compose up -d ancientdata
docker compose logs --tail=80 ancientdata | cat
```

### F) GeoServer `Permission denied` on startup
- GeoServer data directory is writable by host root but not by container UID/GID
- fix and restart:
```bash
sudo mkdir -p /volume1/docker/ancientdata/geoserver
sudo chown -R 1000:1000 /volume1/docker/ancientdata/geoserver
sudo find /volume1/docker/ancientdata/geoserver -type d -exec chmod 775 {} \;
sudo find /volume1/docker/ancientdata/geoserver -type f -exec chmod 664 {} \;
cd /volume1/docker/AncientDataWebGIS
docker compose up -d --force-recreate geoserver
docker logs --tail=120 GeoServer | cat
```

### G) Admin panel "Back up now" fails or status never updates
- see step 6a above (`backup_history` schema) and
  `AncientDataWebGIS/docs/BACKUP-STRATEGY.md`'s Troubleshooting table — most common causes are the
  schema step being skipped, or `BACKUP_NAS_ENABLED`/`BACKUP_DB_ENABLED` left at their default `false`

### H) `git pull --ff-only` blocked by local changes on NAS
- backup + stash local drift, then pull:
```bash
cd /volume1/docker/ancientdataworkspace
cp deploy/nginx/nginx.conf "/tmp/nginx.conf.nas.$(date +%Y%m%d-%H%M%S).bak"
git status --short
git --no-pager diff -- deploy/nginx/nginx.conf | cat
git stash push -m "nas-local-nginx-before-pull" -- deploy/nginx/nginx.conf
git pull --ff-only
```

## 12) Secrets handling

- Never commit real secrets.
- Keep secret names in `.env.example`.
- Store real values in NAS `.env` and a password manager backup.

## 13) WebGIS remediation summary (this cycle)

This cycle codified and validated the following:
- nginx `/webGIS` canonicalization and upstream path handling in repo config
- frontend build/runtime prefix correctness under `/webGIS/`
- backend public GET access for intended guest endpoints
- `/arcade/` trailing-slash proxy fix codified in repo config

Operational learnings captured here:
- verify public API reachability with GET-based checks
- avoid multiple active `ancientdata` or `retrogame` containers on
  `webgis-edge`; keep single owner per service
- treat missing `JWT_SECRET` as a runtime configuration fault, not a code fix
