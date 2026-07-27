# Ancient Data Workspace Deployment

This repository contains only the NAS edge deployment stack for `rinsewillet.net`.
Application source code is intentionally excluded; runtime images are pulled from DockerHub.

## Prerequisites

- NAS host with Docker Engine + Docker Compose plugin installed
- External Docker network already created: `webgis-edge`
- Existing Cloudflare Tunnel: `nas-rinsewillet`
- Cloudflare DNS already delegated to Cloudflare nameservers
- Git access from NAS host

## Deploy / recover on NAS

```bash
git clone https://github.com/RinseWillet/ancientdataworkspace.git
cd ancientdataworkspace/deploy
cp .env.example .env
# edit .env and set CLOUDFLARE_TUNNEL_TOKEN=...
docker compose pull
docker compose up -d --build
```

## Verify

Open:

- https://rinsewillet.net
- https://rinsewillet.net/webGIS/
- https://rinsewillet.net/arcade/

If any route fails, check:

```bash
docker compose ps
docker compose logs --tail=200 cloudflared nginx ancientdata retrogame landing
```

## Router port-forward cleanup

Keep existing router port forwards in place until all three URLs above work correctly through Cloudflare Tunnel.
Remove old port forwards only after successful verification.
