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
mkdir -p /volume1/docker/ancientdata/geoserver
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
```

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

### 8.1) Backend topology models (both supported)

The reverse proxy resolves `ancientdata` on the shared `webgis-edge` network.
You can run `/webGIS/` with either model below, but avoid running both backend
containers at once.

**Model A (recommended operational baseline):**
- Active backend is `AncientDataWebGIS` stack (`ancientdatawebgis-ancientdata-1`)
- `ancientdataworkspace/deploy` stack provides nginx/cloudflared/landing/arcade
- Keep `deploy-ancientdata-1` stopped/removed

**Model B (alternative):**
- Active backend is `ancientdataworkspace/deploy` service (`deploy-ancientdata-1`)
- In this model, ensure required backend env vars (especially `JWT_SECRET`) are
  available to that service runtime

Collision check:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}' | grep ancientdata
```

If both backends are running, nginx may route unpredictably.

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
- for Model A:
```bash
cd /volume1/docker/AncientDataWebGIS
grep -n '^JWT_SECRET=' .env
docker compose up -d ancientdata
docker compose logs --tail=80 ancientdata | cat
```
- for Model B, ensure equivalent env wiring exists in the deploy stack runtime

### F) `git pull --ff-only` blocked by local changes on NAS
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
- avoid multiple active `ancientdata` backends on `webgis-edge` unless model
  ownership is explicit
- treat missing `JWT_SECRET` as a runtime configuration fault, not a code fix
