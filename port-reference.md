# Homelab Service Port Reference

> Internal reference only. Tailscale IPs and credentials are never committed here.

---

## Proxmox Host (<proxmox-ip>)

| Port | Service | Access |
|---|---|---|
| 8006 | Proxmox Web UI | LAN + Tailscale only |
| 22 | SSH | Tailscale only |
| 3128 | SPICE proxy (VM consoles) | LAN only |

---

## CT 101 — docker-services (<ct101-ip>)

| Port | Service | Notes |
|---|---|---|
| 8096 | Jellyfin | Video streaming |
| 32400 | Plex | Video streaming (alt) |
| 4533 | Navidrome | Music |
| 5000 | Kavita | Books / Manga / Comics |
| 13378 | Audiobookshelf | Audiobooks & Podcasts |
| 2283 | Immich | Photos |
| 1234 | Papra | Document management |

All CT101 ports are internal — access via NPM reverse proxy only. Never port-forward directly.

---

## CT 102 — nginx-proxy (<ct102-ip>)

| Port | Service | Notes |
|---|---|---|
| 80 | HTTP | Reverse proxy entry |
| 443 | HTTPS | Reverse proxy + SSL |
| 81 | NPM Admin UI | **LAN only — never expose** |

---

## CT 104 — home-assistant (<ct104-ip>)

| Port | Service | Notes |
|---|---|---|
| 8123 | Home Assistant | Access via Tailscale or NPM |

---

## VM 200 — OPNsense

| Port | Service | Notes |
|---|---|---|
| 443 | Web GUI | After install |
| 22 | SSH | Disabled by default |

---

## Client Apps (no server)

| App | Platform | Connects to |
|---|---|---|
| Symfonium | Android | Navidrome :4533 |
| KOReader | Android / e-ink | Kavita :5000 |

---

## Security Rules

Never forward these ports to the internet via the router:

- `22` — SSH (use Tailscale instead)
- `8006` — Proxmox UI (use Tailscale instead)
- `81` — NPM Admin
- `8123` — Home Assistant (use Tailscale or NPM + auth)
- Any CT101 service port directly — always route through NPM

---

## Access Pattern (correct flow)

```
Internet → OPNsense → NPM (CT102 :80/:443) → Services (CT101)
Remote   → Tailscale → Proxmox / HA directly
```
