# Proxmox Homelab — Dell T1700

Sovereign self-hosted infrastructure running on a single Proxmox VE 9.2.2 node with Tailscale mesh networking and a Corosync QDevice tiebreaker on a Raspberry Pi.

---

## Hardware

| Component | Details |
|---|---|
| Host | Dell Precision T1700 |
| IP | 192.168.1.108 (LAN) |
| Tailscale | 100.x.x.x |
| Storage | ~220 GB LVM-thin pool |
| Quorum | Raspberry Pi (QNetd) |

---

## Network Map

```
Internet
   │
Router (192.168.1.1)
   │
Proxmox T1700 (192.168.1.108)
   ├── CT 101 · docker-services  (192.168.1.109)
   ├── CT 102 · nginx-proxy      (192.168.1.111)  ← ports 80/443/81
   ├── CT 104 · home-assistant   (192.168.1.110)  ← port 8123
   └── VM 200 · opnsense         (install pending)
```

---

## Containers & VMs

### CT 101 — docker-services (192.168.1.109)

Docker host running all media and document services.

| Service | Port | Purpose |
|---|---|---|
| Jellyfin | 8096 | Video streaming |
| Plex | 32400 | Video streaming (alt) |
| Navidrome | 4533 | Music streaming |
| Kavita | 5000 | Books / Comics / Manga |
| Audiobookshelf | 13378 | Audiobooks & Podcasts |
| Immich | 2283 | Photo management |
| Papra | 1234 | Document management |

**Stack location:** `/opt/media-stack/docker-compose.yml`

Restart all services:
```bash
pct exec 101 -- bash -c 'cd /opt/media-stack && docker-compose restart'
```

### CT 102 — nginx-proxy (192.168.1.111)

Nginx Proxy Manager for reverse proxy and SSL termination.

| Port | Purpose |
|---|---|
| 80 | HTTP |
| 443 | HTTPS |
| 81 | NPM Admin UI |

**Admin UI:** `http://192.168.1.111:81`  
Default login: `admin@example.com` / `changeme` (change on first login)

**Stack location:** `/opt/npm/docker-compose.yml`

### CT 104 — home-assistant (192.168.1.110)

Home Assistant Core installed via Python venv.

| Port | Purpose |
|---|---|
| 8123 | HA Web UI |

**Install path:** `/srv/homeassistant`  
**Config path:** `/home/homeassistant/.homeassistant`  
**Run as:** `homeassistant` user

Start/stop:
```bash
pct exec 104 -- bash -c 'systemctl start home-assistant@homeassistant'
pct exec 104 -- bash -c 'systemctl stop home-assistant@homeassistant'
```

### VM 200 — OPNsense

OPNsense 25.1 firewall VM (install from ISO).

| Resource | Value |
|---|---|
| RAM | 2 GB |
| Cores | 2 |
| Disk | 20 GB |
| ISO | OPNsense-25.1.iso |

Start installer from Proxmox web UI → VM 200 → Start.

---

## Infrastructure Services

### Tailscale

Installed on Proxmox host and accessible via Tailscale IP.

```bash
tailscale status
tailscale ip
```

### QDevice (Quorum)

Corosync QDevice running on Raspberry Pi (100.80.155.45) for single-node quorum.

```bash
pvecm status          # check quorum status
pvecm qdevice status  # check QDevice
```

---

## Maintenance

### Daily Cron — Wazuh tmp cleanup (Dell node)

File: `/etc/cron.daily/wazuh-tmp-cleanup`
```bash
#!/bin/bash
docker exec wazuh-manager bash -c \
  'find /var/ossec/queue/vd_updater/tmp/contents/ -type f -mtime +1 -delete' 2>/dev/null
```

### Update all CT 101 containers

```bash
pct exec 101 -- bash -c 'cd /opt/media-stack && docker-compose pull && docker-compose up -d'
```

### Update NPM

```bash
pct exec 102 -- bash -c 'cd /opt/npm && docker-compose pull && docker-compose up -d'
```

---

## Pending / Future

- [ ] OPNsense VM 200 — complete installer via console
- [ ] Configure NPM reverse proxy rules for each service
- [ ] Add NAS node for Syncthing backup when hardware available
- [ ] Enable Proxmox HA when second compute node added
- [ ] Immich external library mount (NAS)
- [ ] SSL certificates via Let's Encrypt in NPM

---

## Port Reference

| IP | Port | Service |
|---|---|---|
| 192.168.1.109 | 8096 | Jellyfin |
| 192.168.1.109 | 32400 | Plex |
| 192.168.1.109 | 4533 | Navidrome |
| 192.168.1.109 | 5000 | Kavita |
| 192.168.1.109 | 13378 | Audiobookshelf |
| 192.168.1.109 | 2283 | Immich |
| 192.168.1.109 | 1234 | Papra |
| 192.168.1.110 | 8123 | Home Assistant |
| 192.168.1.111 | 81 | NPM Admin |
| 192.168.1.111 | 80/443 | Reverse Proxy |
