# Proxmox Homelab Services Stack
> Self-hosted media, photo, document, and home automation stack deployed on Proxmox VE with High Availability

---

## Services Overview

| Service | Purpose | Type | Port |
|---|---|---|---|
| [Jellyfin](https://jellyfin.org/) | Media server (movies, TV, music) | LXC | 8096 |
| [Navidrome](https://www.navidrome.org/) | Music streaming server | LXC | 4533 |
| [Audiobookshelf](https://www.audiobookshelf.org/) | Audiobook & podcast server | LXC | 13378 |
| [Kavita](https://www.kavitareader.com/) | Manga / Comics / Books reader | LXC | 5000 |
| [Immich](https://immich.app/) | Photo & video backup (Google Photos alternative) | LXC | 2283 |
| [Papra](https://papra.app/en/) | Document management | LXC | 1300 |
| [Home Assistant](https://www.home-assistant.io/) | Home automation | VM | 8123 |

> **Note:** Symfonium and KOReader are client apps (mobile/desktop) — no hosting required.
> Plex is an alternative to Jellyfin. Choose one.

---

## Architecture

### Node Setup (Proxmox Cluster)

```
Node 1: Dell Precision T1700 (16GB RAM, 512GB SSD)
Node 2: [your second node]
Node 3: [your third node — required for HA quorum]
```

> ⚠️ Proxmox High Availability requires a minimum of **3 nodes** for quorum.
> With only 2 nodes, split-brain scenarios cannot be resolved automatically.

### Storage Strategy

```
Local storage  → OS and container templates
Shared storage → All HA containers/VMs (NFS, iSCSI, or Ceph)
Media storage  → Large HDD for Jellyfin, Immich, Audiobookshelf
```

---

## RAM Allocation Plan (per node)

| Container | RAM | vCPU | Storage |
|---|---|---|---|
| Jellyfin | 2GB | 2 | 20GB + media mount |
| Navidrome | 512MB | 1 | 10GB + music mount |
| Audiobookshelf | 512MB | 1 | 10GB + books mount |
| Kavita | 512MB | 1 | 10GB + library mount |
| Immich | 2GB | 2 | 20GB + photo mount |
| Papra | 512MB | 1 | 10GB |
| Home Assistant | 2GB | 2 | 32GB |
| **Total** | **~8GB** | **10** | |

---

## High Availability Configuration

### Requirements
- Minimum 3 Proxmox nodes in a cluster
- Shared storage accessible by all nodes (NFS, Ceph, or iSCSI)
- Corosync network (dedicated NIC recommended)
- Fencing device or QEMU guest agent

### Enable HA on a container/VM
```bash
# In Proxmox web UI:
# Datacenter > HA > Add resource
# Select container ID > Set state to "started"
# Set max restart = 3, max relocate = 3
```

### Recommended HA Groups

| Group | Priority | Members |
|---|---|---|
| media | High | Jellyfin, Navidrome, Audiobookshelf |
| storage | High | Immich, Kavita, Papra |
| automation | Critical | Home Assistant |

---

## Deployment Steps

### 1. Create the Proxmox Cluster
```bash
# On Node 1 (primary)
pvecm create my-homelab-cluster

# On Node 2 and 3
pvecm add <node1-ip>

# Verify cluster status
pvecm status
```

### 2. Set up Shared Storage
```bash
# Example: NFS shared storage
# Datacenter > Storage > Add > NFS
# ID: shared-nfs
# Server: <nfs-server-ip>
# Export: /mnt/shared
# Enable: all nodes
```

### 3. Deploy LXC Containers on Shared Storage
```bash
# Example: Jellyfin container
pct create 100 local:vztmpl/debian-12-standard_12.2-1_amd64.tar.zst \
  --hostname jellyfin \
  --memory 2048 \
  --cores 2 \
  --storage shared-nfs \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp
```

### 4. Enable HA for Each Container
```bash
# Via CLI
ha-manager add ct:100 --state started --max_restart 3 --max_relocate 3
ha-manager add ct:101 --state started --max_restart 3 --max_relocate 3
# ... repeat for each service
```

---

## Network Layout

```
<proxmox-ip>  →  Proxmox Web UI (port 8006)
192.168.1.x    →  Jellyfin      (port 8096)
192.168.1.x    →  Navidrome     (port 4533)
192.168.1.x    →  Audiobookshelf (port 13378)
192.168.1.x    →  Kavita        (port 5000)
192.168.1.x    →  Immich        (port 2283)
192.168.1.x    →  Papra         (port 1300)
192.168.1.x    →  Home Assistant (port 8123)
```

> Assign static IPs to each container via DHCP reservation or static config.

---

## Client Apps

| App | Platform | Connects to |
|---|---|---|
| [Symfonium](https://symfonium.app/) | Android | Navidrome |
| [KOReader](https://koreader.rocks/) | Android / e-ink | Kavita |
| Jellyfin mobile app | iOS / Android | Jellyfin |
| Immich mobile app | iOS / Android | Immich |

---

## Author

**Dinaverse** — [GitHub](https://github.com/Dinaverse) | [LinkedIn](https://www.linkedin.com/in/dinacima)
