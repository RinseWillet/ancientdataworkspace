# Deploying rinsewillet.net on the Synology NAS

> **Canonical guide:** for complete rebuild/recovery instructions and troubleshooting from the latest cutover, see `../docs/deployment-recovery.md`.
>
> This README is kept as a quick reference; the full runbook is the source of truth.

Single public entry point (`rinsewillet.net`), fronted by a free Cloudflare
Tunnel + an `nginx` reverse proxy, routing to three internal apps:

| Path | App | Repo |
|---|---|---|
| `/` | Landing page | `ancientdataworkspace/deploy/landing-page` (this repo) |
| `/webGIS/` | AncientDataWebGIS | `AncientDataWebGIS` + `AncientDataWebGIS_FE` |
| `/arcade/` | RetroGamesApp | `RetroGameApp` (external repo) |

No ports are forwarded on the KPN Experia Box for any of this — `cloudflared`
only makes outbound connections to Cloudflare's edge. **Cost: $0** (Cloudflare
Tunnel + free plan DNS/proxy/WAF/Access are all free for personal use).

---

## 0. Prerequisites

- A free [Cloudflare account](https://dash.cloudflare.com/sign-up).
- Container Manager on the NAS, with the ability to run `docker` CLI over SSH
  (Control Panel → Terminal & SNMP → enable SSH) — easiest way to run the
  one-off `docker network create` command and `docker compose` commands below.
- `rinsewillet.net` registrar access, to change nameservers.

## 1. Point the domain at Cloudflare

1. **Sign up / log in** at [dash.cloudflare.com](https://dash.cloudflare.com) — free, no card required.
2. **Add a domain**: dashboard → **Add a domain** (or **+ Add** in the account
   picker) → type `rinsewillet.net` → **Continue**.
3. **Select the Free plan** on the plan-selection screen → **Continue**.
4. **DNS record scan**: Cloudflare automatically scans for any existing DNS
   records at your current registrar/host and shows them for review.
   - If this is a brand-new domain with no existing site/mail, you likely
     have nothing important yet — you can review and continue; you don't
     need to pre-create an A record for `rinsewillet.net` since the Tunnel
     (step 3) will add the routing record automatically.
   - If you already use email on this domain (e.g. MX records), **make sure
     those are imported/kept** so email keeps working.
5. Cloudflare now shows **two nameservers**, e.g.:
   ```
   ns1.cloudflare.com
   ns2.cloudflare.com
   ```
   (Yours will be different — Cloudflare assigns a pair per zone.)
6. **Log in to your domain registrar** (wherever you bought `rinsewillet.net`
   — commonly TransIP, Versio, Namecheap, or GoDaddy) and find the
   **nameserver / DNS management** section for the domain (often called
   "Nameservers", "DNS settings", or "Name server change"). Replace the
   existing nameservers with the two Cloudflare gave you. Remove any other
   nameserver entries — Cloudflare should be the *only* pair listed.
   - ⚠️ Don't confuse this with adding a "DNS record" at the registrar — you
     are changing which DNS provider is authoritative for the whole domain.
   - If your registrar has "DNSSEC" enabled, disable it first or you'll need
     to update the DS record afterwards per Cloudflare's DNSSEC setup page
     (Cloudflare will prompt you if this applies).
7. **Wait for activation.** Cloudflare emails you once it detects the
   nameserver change (often 15 min–a few hours, rarely up to 24h). You can
   check progress yourself without waiting for email:
   ```bash
   dig NS rinsewillet.net +short
   ```
   or check https://www.whatsmydns.net/#NS/rinsewillet.net — once most
   locations show the Cloudflare nameservers, the zone is active.
8. Once the Cloudflare dashboard shows the zone status as **Active**, you're
   done with this step — proceed to step 3 (Create the Tunnel), which will
   add the actual `rinsewillet.net` DNS record pointing into the tunnel. You
   do not need to manually create any A/CNAME record yourself.

## 2. Create the shared Docker network (once)

This network lets containers from separate Compose projects
(`AncientDataWebGIS`, `RetroGameApp`, and this `deploy/` stack) find each
other by service name, without publishing any ports to the NAS's LAN/WAN
interfaces.

**Option A — Container Manager GUI (no SSH needed):**
1. Open **Container Manager** on the NAS → left sidebar → **Network**.
2. Click **Add** → name it exactly `webgis-edge` → driver **bridge** → leave
   other settings default → **Apply**.
3. This network now shows up in every project's "Network" dropdown when you
   create/edit a project in Container Manager.

**Option B — SSH/CLI (equivalent, useful for scripting):**
1. Control Panel → **Terminal & SNMP** → check **Enable SSH service** → Apply.
2. From your computer:
   ```bash
   ssh <your-admin-user>@<nas-ip> -p 22
   ```
3. Create the network:
   ```bash
   sudo docker network create webgis-edge
   ```
   (On DSM 7.2+ with Container Manager, the `docker` CLI is available at
   `/usr/local/bin/docker`; if `docker` isn't found, try
   `/usr/local/bin/docker network create webgis-edge`.)
4. Verify it exists:
   ```bash
   sudo docker network ls | grep webgis-edge
   ```

Either option produces the same result — the network persists across reboots
and container/stack redeploys, so this is a genuine one-time step. If you
ever tear down and rebuild everything from scratch, just re-run this step
before bringing up any of the three Compose stacks.

## 3. Create the Tunnel

1. Go to the **Zero Trust dashboard**: from the main Cloudflare dashboard,
   left sidebar → **Zero Trust** (or directly at
   [one.dash.cloudflare.com](https://one.dash.cloudflare.com)).
   - **First time only:** Cloudflare will ask you to choose a **team name**
     (any unique slug, e.g. `rinsewillet`) and select the **Free** plan
     (up to 50 users) — it may also ask for a phone number for verification.
     None of this costs anything and doesn't affect the main site config.
2. Left sidebar → **Networks → Tunnels** → **Create a tunnel**.
3. Choose connector type **Cloudflared** → **Next**.
4. Name the tunnel, e.g. `nas-rinsewillet` → **Save tunnel**.
5. You're now on the **"Install and run a connector"** screen, showing
   commands for various platforms (Docker, Windows, macOS, Linux). We don't
   need to run any of them manually — just **copy the token** (the long
   string after `--token` in the shown `docker run cloudflare/cloudflared ...
   --token <TOKEN>` command). Paste it into `deploy/.env`:
   ```dotenv
   CLOUDFLARE_TUNNEL_TOKEN=<paste the token here>
   ```
   (copy `.env.example` → `.env` first if you haven't already). The
   `cloudflared` service in `docker-compose.yml` already references
   `${CLOUDFLARE_TUNNEL_TOKEN}`, so once this container starts it will
   register itself as this tunnel's connector — no separate install needed.
6. Click **Next** (don't worry that the connector isn't "seen" yet — it'll
   show as connected once you actually run `docker compose up -d` in step 4).
7. On the **Public Hostname** screen, add one route:
   - **Subdomain**: leave blank
   - **Domain**: `rinsewillet.net`
   - **Path**: leave blank
   - **Type**: `HTTP`, **URL**: `nginx:80`
   
   Click **Save tunnel**. This is the step that automatically creates the
   correct DNS record (a CNAME to `<tunnel-id>.cfargotunnel.com`, proxied) —
   you don't need to touch the DNS records tab again for this.
8. Back in the main Cloudflare dashboard (not Zero Trust) for
   `rinsewillet.net` → **SSL/TLS** → **Overview**, set encryption mode to
   **Full** (traffic between Cloudflare and `cloudflared` is already
   authenticated/encrypted by the tunnel connection itself, so Full is
   sufficient — no origin certificate needs to be installed anywhere).
9. The tunnel will show status **"Down"/gray** in the Tunnels list until you
   actually start the `cloudflared` container (step 4 below) — that's
   expected at this point, not an error.

## 4. Deploy the app stacks (Container Manager → Project, or CLI)

Order matters (dependencies use container-name DNS resolution):

```bash
# 1. Backend + GeoServer
cd AncientDataWebGIS
docker compose up -d
```

In `AncientDataWebGIS/.env`, set these for the new domain/runtime (minimum set shown below; full set in the runbook):

```dotenv
DB_URL=jdbc:postgresql://192.168.2.13:2665/webGIS_DB
DB_USER=root
DB_PASSWORD=<db-password>
JWT_SECRET=<long-random-secret>
JWT_EXPIRATION=86400000
MEDIA_BASE_URL=https://rinsewillet.net/webGIS/api/media/files
CORS_ALLOWED_ORIGINS=https://rinsewillet.net
```

```bash
# 2. Arcade app — see "RetroGamesApp integration" below for its compose file
cd ../RetroGameApp
docker compose pull
docker compose up -d --no-build

# 3. Reverse proxy + tunnel + landing page
cd ../ancientdataworkspace/deploy
cp .env.example .env   # fill in CLOUDFLARE_TUNNEL_TOKEN
docker compose pull
docker compose up -d --no-build
```


Visit `https://rinsewillet.net`, `https://rinsewillet.net/webGIS/`, and
`https://rinsewillet.net/arcade/` — all served over HTTPS automatically via
Cloudflare's edge certificate.

## 5. RetroGamesApp integration (React + Vite)

In the `RetroGameApp` repo:

1. Set the Vite base path so built asset URLs resolve correctly under
   `/arcade/`:
   ```js
   // vite.config.js
   export default defineConfig({
     base: process.env.VITE_BASE_PATH || '/',
     // ...
   })
   ```
2. Build with `VITE_BASE_PATH=/arcade/ npm run build` (in its Dockerfile/CI).
3. Add a small static-serving Dockerfile (same pattern as
   `deploy/landing-page/Dockerfile`):
   ```dockerfile
   FROM nginx:1.27-alpine
   COPY dist/ /usr/share/nginx/html/
   EXPOSE 80
   ```
4. Add a `docker-compose.yml` in that repo:
   ```yaml
   services:
     retrogame:
       image: rinsedev/retrogameapp:latest
       restart: unless-stopped
       expose:
         - "80"
       networks:
         - webgis-edge
   networks:
     webgis-edge:
       external: true
   ```
   The service **must** be named `retrogame` (or update `nginx.conf`'s
   `/arcade/` `proxy_pass` to match) so `nginx` can resolve it by name.

## 6. Close the old exposure on the Experia Box

Once step 4 is confirmed working end-to-end over `https://rinsewillet.net`,
log into the Experia Box admin UI and **delete every port-forwarding rule**
that currently exists for this NAS:

- WebGIS (old direct port) — replaced by `/webGIS/` via the tunnel.
- RetroGamesApp (old direct HTTP port) — replaced by `/arcade/` via the tunnel.
- GeoServer, PostGISDB, PGAdmin — **do not forward these on the router at
  all.** See section 8 below for how you still get secure remote access to
  these (for QGIS, GeoServer admin, etc.) without any port-forward.

After this, the NAS should have **zero** inbound port-forwards on the router;
`cloudflared`'s outbound-only connection is the sole path in for the website,
and Cloudflare WARP (section 8) is the sole path in for DB/GeoServer.

## 7. (Optional, free) Gate the admin panel with Cloudflare Access

Zero Trust → **Access → Applications** → **Add an application** → **Self-hosted**:
- Domain: `rinsewillet.net`, Path: `/webGIS/admin-panel`
- Policy: allow only your own email (one-time PIN or GitHub/Google login).

This blocks unauthenticated requests to the admin panel at Cloudflare's edge,
before they ever reach the container — free on the standard plan, no app
changes needed.

## 8. Database & GeoServer access (QGIS, admin UIs)

Unlike the website, GeoServer/Postgres/PGAdmin are **not** meant to be public
at all — but you still want to reach them yourself, from QGIS, potentially
from outside your home network. Two situations:

### At home, on the same LAN — no tunnel needed
`geoserver` publishes a **host** port (`ports: - "2675:8080"` in
`AncientDataWebGIS/docker-compose.yml`) and Postgres/PGAdmin do the same in
their own stack (the separate `postgis_pgadmin_stack`, run outside this
workspace). "Host port" ≠ "internet exposure" — it just means the NAS's LAN
IP (e.g. `192.168.1.50`) answers on that port for anything on your home
network. As long as the *router* doesn't forward it externally (section 6),
QGIS on your home Wi-Fi/LAN connects directly to
`192.168.1.50:2675` (GeoServer) or `192.168.1.50:2665` (Postgres) — nothing
extra to set up.

### Away from home — reuse the same Cloudflare Tunnel via WARP (free)
Cloudflare Zero Trust Tunnels support **Private Network routing**: instead of
(or in addition to) the HTTP "Public Hostname" we set up for the website, you
can route an entire private IP range through the tunnel, then reach it from
any device running the **Cloudflare WARP** client, logged into your Zero
Trust org. No VPN server, no extra ports, no extra cost.

1. Zero Trust dashboard → **Networks → Tunnels** → click `nas-rinsewillet` →
   **Private Network** tab → **Add a private network**.
2. Enter the NAS's LAN CIDR, e.g. `192.168.1.50/32` (just the NAS itself —
   safest) or your whole LAN subnet (e.g. `192.168.1.0/24`) if you want
   broader reach.
3. Install the **Cloudflare WARP** client on your laptop/desktop (the one
   running QGIS): [one.dash.cloudflare.com](https://one.dash.cloudflare.com) →
   your team name → download WARP for your OS → log in with your Zero Trust
   account → enable it (it runs as a lightweight background client/VPN-style
   connector, not a full network VPN — only traffic to routed private IPs
   goes through the tunnel, everything else is unaffected).
4. In QGIS's PostGIS connection dialog, connect to the NAS's **LAN IP**
   directly (e.g. `192.168.1.50`, port `2665`) — WARP transparently routes
   this through the tunnel when you're off-network, and it's simply local
   traffic when you're on the home LAN. Same for GeoServer's admin UI in a
   browser (`http://192.168.1.50:2675/geoserver`) or PGAdmin.

This means: **one Tunnel, one free Cloudflare account**, serving both the
public website (`rinsewillet.net/webGIS`, `/arcade`, `/`) and your own
private, encrypted, no-port-forward access to the database/GeoServer/PGAdmin
— consistent with the "don't want to start paying later" requirement, since
Private Network routing on Cloudflare Tunnels is free for personal accounts.

### Alternative if you'd rather not install WARP
Synology's **VPN Server** package (WireGuard, free, built into DSM) is a
perfectly good alternative for this specific use case — connect QGIS's
machine to the NAS via WireGuard, then use the NAS's LAN IP the same way.
Slightly more setup (key exchange, DSM package install) but keeps things
entirely within Synology rather than relying on Cloudflare for this part too.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| 502 from nginx on `/webGIS/` | `ancientdata` container not on `webgis-edge`, or not yet healthy |
| `/webGIS/api/...` returns `401` when checked with `curl -I` | `curl -I` uses `HEAD`; re-test with `GET` (`curl -sS -o /dev/null -w "%{http_code}\n" ...`) |
| Assets 404 under `/webGIS/` | Frontend built without `VITE_BASE_PATH=/webGIS/` (check the Docker image build args in `docker-image.yml`) |
| API calls hit `/api/...` instead of `/webGIS/api/...` | `VITE_API_BASE_URL` missing at frontend build time |
| Intermittent `401/502` on WebGIS after stack updates | Multiple active `ancientdata` containers on `webgis-edge` causing ambiguous backend routing |
| `deploy-ancientdata-1` restart loop with `JwtUtil`/missing secret error | Runtime `JWT_SECRET` missing in the backend container environment |
| Tunnel shows "down" in Zero Trust dashboard | `cloudflared` container not running / bad `CLOUDFLARE_TUNNEL_TOKEN` |
| Can't reach GeoServer/Postgres via WARP from outside | Private Network CIDR not added to the tunnel, or WARP client not logged into the right Zero Trust team |
| Arcade loads HTML for JS/CSS (`MIME type text/html`) | `/arcade/` nginx proxy missing trailing slash on `proxy_pass`, or stale Cloudflare cache for `/arcade/*` |

### Critical operating notes

- For full WebGIS recovery and reproducibility procedures, use
  `../docs/deployment-recovery.md` as source of truth.
- `/webGIS/` API validation should use **GET-based** status checks.
- If you run multiple Compose stacks on `webgis-edge`, ensure backend ownership
  is explicit (avoid two active `ancientdata` services unless intentionally
  managed).
