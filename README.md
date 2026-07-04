# Proxmox Homelab — Dell T1700

Sovereign self-hosted infrastructure running on a single Proxmox VE 9.2.2 node with Tailscale mesh networking and a Corosync QDevice tiebreaker on a Raspberry Pi.

---

## Hardware

| Component | Details |
|---|---|
| Host | Dell Precision T1700 |
| IP | `<proxmox-lan-ip>` (static LAN, set in router) |
| Remote access | Tailscale VPN (no IP published) |
| Storage | ~220 GB LVM-thin pool |
| Quorum | Raspberry Pi (QNetd via Tailscale) |

---

## Network Map

```
Internet
   │
Router
   │
Proxmox T1700
   ├── CT 101 · docker-services   ← media stack
   ├── CT 102 · nginx-proxy       ← ports 80/443 (public), 81 (localhost only)
   ├── CT 104 · home-assistant    ← port 8123 (LAN/Tailscale only)
   └── VM 200 · opnsense          ← firewall
```

---

## Containers & VMs

### CT 101 — docker-services

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

### CT 102 — nginx-proxy

Nginx Proxy Manager for reverse proxy and SSL termination.

| Port | Purpose |
|---|---|
| 80 | HTTP |
| 443 | HTTPS |
| 81 | NPM Admin UI — **localhost only, never expose** |

Access the NPM admin UI via SSH tunnel:
```bash
ssh -L 8081:127.0.0.1:81 user@<ct102-ip>
# then open http://localhost:8081
```

**Stack location:** `/opt/npm/docker-compose.yml`

### CT 104 — home-assistant

Home Assistant Core installed via Python venv.

| Port | Purpose |
|---|---|
| 8123 | HA Web UI (LAN / Tailscale only) |

**Install path:** `/srv/homeassistant`  
**Config path:** `/home/homeassistant/.homeassistant`

Start/stop:
```bash
pct exec 104 -- bash -c 'systemctl start home-assistant@homeassistant'
pct exec 104 -- bash -c 'systemctl stop home-assistant@homeassistant'
```

### VM 200 — OPNsense

OPNsense 25.1 firewall VM.

| Resource | Value |
|---|---|
| RAM | 2 GB |
| Cores | 2 |
| Disk | 20 GB |

---

## Infrastructure Services

### Tailscale

Installed on Proxmox host. All remote management goes through Tailscale — no ports forwarded to internet.

```bash
tailscale status
tailscale ip
```

### QDevice (Quorum)

Corosync QDevice running on Raspberry Pi for single-node quorum tiebreaker.

```bash
pvecm status          # check quorum status
pvecm qdevice status  # check QDevice
```

---

## Maintenance

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

- [ ] OPNsense — finish install and configure WAN/LAN
- [ ] Configure NPM reverse proxy rules for each service
- [ ] NAS node for Syncthing backup when hardware available
- [ ] Proxmox HA when second compute node added
- [ ] Immich external library mount (NAS)
- [ ] SSL certificates via Let's Encrypt in NPM

---

## Port Reference

| Port | Service | Accessible from |
|---|---|---|
| 8096 | Jellyfin | LAN via NPM |
| 32400 | Plex | LAN via NPM |
| 4533 | Navidrome | LAN via NPM |
| 5000 | Kavita | LAN via NPM |
| 13378 | Audiobookshelf | LAN via NPM |
| 2283 | Immich | LAN via NPM |
| 1234 | Papra | LAN via NPM |
| 8123 | Home Assistant | LAN / Tailscale |
| 80/443 | NPM reverse proxy | LAN (+ internet if DNS configured) |
| 8006 | Proxmox Web UI | LAN / Tailscale only |
| 22 | SSH | Tailscale only |

---

## Security Model

- All remote access via Tailscale — zero ports forwarded to internet
- NPM admin (port 81) bound to localhost only — SSH tunnel required
- Proxmox firewall enabled: SSH and web UI restricted to LAN + Tailscale
- rpcbind disabled (was port 111)
