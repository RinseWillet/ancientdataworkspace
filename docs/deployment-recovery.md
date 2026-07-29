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

## 9) Validation checks

```bash
curl -I https://rinsewillet.net/
curl -I https://rinsewillet.net/webGIS/
curl -I https://rinsewillet.net/arcade/
```

Expected:
- `/` -> `200`
- `/arcade/` -> `200`
- `/webGIS/` -> reachable through backend (application may return `401` for unauthenticated paths; this is app-level, not tunnel failure)

Internal service checks:

```bash
cd /volume1/docker/ancientdataworkspace/deploy
docker compose exec nginx wget -S -O- http://landing:80/ 2>&1 | head -n 20
docker compose exec nginx wget -S -O- http://ancientdata:8080/ 2>&1 | head -n 20
docker compose exec nginx wget -S -O- http://retrogame:80/ 2>&1 | head -n 20
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

## 12) Secrets handling

- Never commit real secrets.
- Keep secret names in `.env.example`.
- Store real values in NAS `.env` and a password manager backup.

## 13) Prompt to start WebGIS functional remediation

Use this prompt in a fresh implementation session:

```text
Context:
- Infrastructure is now working: Cloudflare Tunnel -> nginx -> containers.
- Public routes: `/` works, `/arcade/` works.
- WebGIS backend is reachable through `/webGIS/`, but functional parity is incomplete.

Goal (phase 1, required):
- As an unauthenticated guest, I can open `https://rinsewillet.net/webGIS/` and use the public-facing map UI without blank pages, broken assets, or gateway/proxy errors.

Current architecture:
- Domain: rinsewillet.net
- Reverse proxy: ancientdataworkspace/deploy/nginx/nginx.conf
- Backend image: rinsedev/ancientdata:latest
- Runtime vars are provided via AncientDataWebGIS/.env and compose env pass-through.

Please do:
1) Trace the complete request/asset path for `/webGIS/` (HTML, JS, CSS, API calls).
2) Identify whether failures are due to base path handling, auth guard defaults, CORS/security filter behavior, or frontend runtime routing.
3) Propose minimal repo changes to make guest access to public WebGIS pages work.
4) Implement those changes in repo code/config (not ad hoc NAS-only changes).
5) Provide exact verification commands and expected responses.

Constraints:
- Keep pull-only deployment model (Git + DockerHub images).
- Do not weaken security broadly; scope access to intended public guest flows only.
- Keep `/arcade/` and root landing behavior unchanged.
```

