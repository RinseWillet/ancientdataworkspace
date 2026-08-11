# NAS Power Recovery & Auto-Restart Guide

**Last Updated:** August 5, 2026  
**Status:** ✅ Tested and Verified

## Overview

This guide documents the **automatic recovery system** for rinsewillet.net when the NAS loses power and reboots. The system automatically brings all Docker containers back online in the correct dependency order, ensuring the website is accessible without manual intervention.

---

## What Happens When Power Is Lost

1. **Power outage** → NAS shuts down unexpectedly
2. **Power restored** → NAS boots up normally
3. **Docker daemon starts** (standard Synology DSM service)
4. **docker-startup.service** runs automatically (our custom systemd service)
5. **All containers restart in dependency order**
6. **Website is online** within ~2-3 minutes

---

## Architecture: Four Docker Compose Stacks

Your deployment uses **four separate compose projects** that must start in a specific order to avoid connection errors:

### Stack 1: Infrastructure (Database & Admin)
- **Location:** `/volume1/docker/ancientdata/`
- **Services:** PostGIS (database), pgAdmin
- **Starts:** First (other stacks depend on database)
- **Wait condition:** Database healthcheck (`pg_isready`)

### Stack 2: Web Application Backend
- **Location:** `/volume1/docker/AncientDataWebGIS/`
- **Services:** AncientData Java app, GeoServer
- **Starts:** Second (depends on PostGIS)
- **Wait condition:** 10-second pause to allow app initialization

### Stack 3: Arcade Application
- **Location:** `/volume1/docker/RetroGameApp/`
- **Services:** retrogame app
- **Starts:** Third (depends on network + backend routing layer expectation)

### Stack 4: Public-Facing Reverse Proxy & Tunnel
- **Location:** `/volume1/docker/ancientdataworkspace/deploy/`
- **Services:** nginx (reverse proxy), cloudflared (Tunnel), landing page
- **Starts:** Fourth (depends on backend apps being ready)
- **Final step:** Restart nginx to refresh connection pool

---

## Auto-Restart Configuration

### systemd Service File
**Location:** `/etc/systemd/system/docker-startup.service`

```ini
[Unit]
Description=Docker Compose Startup Orchestrator for rinsewillet.net
After=docker.service
Wants=docker.service

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/docker-startup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

**Key properties:**
- `After=docker.service` — Waits for Docker daemon to start
- `Type=oneshot` — Runs once at boot and exits cleanly
- `RemainAfterExit=yes` — Marks as successful after first run
- `WantedBy=multi-user.target` — Runs in normal multi-user boot mode

### Startup Script
**Location:** `/usr/local/bin/docker-startup.sh`

The script:
1. Creates the `webgis-edge` Docker network (if missing)
2. Starts infrastructure stack (PostGIS + pgAdmin)
3. Polls database with `pg_isready` (up to 30 attempts, 2 seconds between)
4. Starts AncientDataWebGIS stack (ancientdata + GeoServer)
5. Waits 10 seconds for app initialization
6. Starts RetroGameApp stack (`retrogame`)
7. Starts deploy stack (nginx + cloudflared + landing)
8. Restarts nginx to refresh backend connection pool
9. Logs all output to `/var/log/docker-startup.log`

**Execution time:** ~60-90 seconds total

---

## Verification & Monitoring

### Check Service Status
```bash
ssh Antoninus138CE@192.168.2.13

# View service status
sudo systemctl status docker-startup.service

# View detailed logs
tail -100 /var/log/docker-startup.log

# Check all containers are running
docker ps -a
```

### Test the Website
After recovery, verify all three services:
```bash
curl -I https://rinsewillet.net/                # Landing page
curl -I https://rinsewillet.net/webGIS/         # Web GIS app
curl -I https://rinsewillet.net/arcade/         # Retro game app
```

All should return `HTTP/1.1 200 OK` (or redirect codes for Cloudflare).

---

## Manual Recovery (If Needed)

If the automatic recovery doesn't work, manually restart services:

```bash
ssh Antoninus138CE@192.168.2.13

# Manually trigger the startup script
sudo /usr/local/bin/docker-startup.sh

# Monitor the log
tail -f /var/log/docker-startup.log
```

---

## Testing the Recovery System

To simulate a power loss and verify auto-restart works:

### Simulated Power Failure Test
```bash
ssh Antoninus138CE@192.168.2.13

# Stop all containers (simulating unclean shutdown)
docker-compose -f /volume1/docker/ancientdata/docker-compose.yml down
docker-compose -f /volume1/docker/AncientDataWebGIS/docker-compose.yml down
docker-compose -f /volume1/docker/RetroGameApp/docker-compose.yml down
docker-compose -f /volume1/docker/ancientdataworkspace/deploy/docker-compose.yml down

# Wait 5 seconds
sleep 5

# Manually trigger the startup script (simulating systemd boot)
sudo /usr/local/bin/docker-startup.sh

# Verify all containers came back online
docker ps
```

**Expected result:** All containers running, website fully operational.

---

## Troubleshooting

### Issue: Website still shows 502 Bad Gateway after recovery

**Cause:** nginx lost connection to backend before it was ready.

**Solution:**
```bash
# Restart just the nginx container
cd /volume1/docker/ancientdataworkspace/deploy/
docker-compose restart nginx

# Check nginx logs
docker-compose logs nginx
```

### Issue: WebGIS app loads but shows connection error

**Cause:** AncientData backend can't connect to PostGIS database.

**Solution:**
```bash
# Check if PostGIS is running and healthy
docker ps | grep PostGIS

# Check database logs
docker logs PostGIS

# Verify database connectivity (from NAS shell)
docker exec PostGIS pg_isready -d webGIS_DB -U root
```

### Issue: Startup script fails silently

**Solution:** Check the log file:
```bash
tail -200 /var/log/docker-startup.log
```

Common issues logged:
- Network creation failed (usually harmless)
- Container name conflicts (pgAdmin legacy container) — safe to ignore
- Database didn't become healthy in time — may need to increase wait timeout

### Issue: Service never runs at boot

**Solution:** Verify systemd configuration:
```bash
# Check if service is enabled
sudo systemctl is-enabled docker-startup.service

# List boot order dependencies
sudo systemctl list-dependencies docker-startup.service

# Re-enable if needed
sudo systemctl enable docker-startup.service
```

---

## Files Reference

| Path | Purpose |
|------|---------|
| `/etc/systemd/system/docker-startup.service` | systemd service definition |
| `/usr/local/bin/docker-startup.sh` | Startup orchestration script |
| `/var/log/docker-startup.log` | Startup logs (readable by all users) |
| `/volume1/docker/ancientdata/` | Infrastructure compose (PostGIS, pgAdmin) |
| `/volume1/docker/AncientDataWebGIS/` | WebGIS backend compose |
| `/volume1/docker/RetroGameApp/` | Arcade compose (`retrogame`) |
| `/volume1/docker/ancientdataworkspace/deploy/` | Public-facing stack (nginx, cloudflared) |

---

## Future Enhancements

Possible improvements (not yet implemented):

1. **Health check notifications** — Email alert if recovery fails
2. **Metrics collection** — Track how often recoveries happen
3. **Automated testing** — Weekly simulated power-loss test
4. **Pre-boot validation** — Check disk space, network status before starting
5. **Database backup verification** — Ensure latest backup is accessible on recovery

---

## Related Documentation

- **Deployment architecture:** `ancientdataworkspace/deploy/README.md`
- **Full recovery runbook:** `ancientdataworkspace/docs/deployment-recovery.md`
- **Docker Compose files:** See individual project directories
- **Infrastructure setup:** `AncientDataWebGIS/docs/ci-cd/docker-infra-compose.yml`

---

## Recovery History

| Date | Event | Resolution |
|------|-------|-----------|
| 2026-08-05 | Power outage | Manual recovery + automated system configured |

---

## Support & Questions

If you need to modify the recovery system:
1. Edit `/usr/local/bin/docker-startup.sh` on the NAS
2. Test with: `sudo /usr/local/bin/docker-startup.sh`
3. Check logs: `tail -f /var/log/docker-startup.log`
4. Reload systemd if service definition changes: `sudo systemctl daemon-reload`

---

**✅ System verified working:** August 5, 2026, 21:27 UTC
