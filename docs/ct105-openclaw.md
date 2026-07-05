# CT105 — OpenClaw AI Gateway Setup

**Date:** 2026-07-05  
**Author:** Dina Cima Mufungizi

---

## Overview

OpenClaw is a self-hosted, always-on AI agent gateway running on Proxmox LXC
container CT105 (hostname: `openclaw`). It uses the Qwen 3.5 27B model served
locally by Ollama on the Arch Linux machine — no cloud API key required.

---

## Infrastructure

| Role | Host | Network |
|------|------|---------|
| Proxmox Host | `dina` (Proxmox node) | Tailscale mesh |
| CT105 | `openclaw-ct105` | Tailscale mesh |
| Ollama server | `archlinux` | Tailscale mesh |

All communication is over the Tailscale VPN mesh. No ports exposed to the internet.

---

## CT105 Specs

| Setting | Value |
|---------|-------|
| ID | 105 |
| Hostname | openclaw |
| OS | Ubuntu 24.04 LTS |
| CPU | 2 vCPU |
| RAM | 2048 MB |
| Swap | 512 MB |
| Disk | 20 GB (local-lvm) |
| Features | nesting=1 |
| Onboot | yes |

---

## LXC Extra Config

File: `/etc/pve/lxc/105.conf`

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Required for Tailscale TUN device access inside the unprivileged container.

---

## Installed Software

- Node.js 22 (NodeSource apt repo)
- OpenClaw 2026.6.11 (`npm install -g openclaw`)
- Tailscale 1.98.8

---

## Ollama Setup (Arch Linux node)

**Model:** `qwen3.5:27b` — 27.8B parameters, Q4_K_M quantization, 17 GB  
**GPU backend:** Vulkan (`OLLAMA_VULKAN=1`)

By default Ollama binds to `127.0.0.1` only. A systemd drop-in override makes
it listen on the Tailscale interface so CT105 can reach it across the mesh.

**Override file:** `/etc/systemd/system/ollama.service.d/tailscale.conf`

```ini
[Service]
Environment="OLLAMA_HOST=<ARCH-TAILSCALE-IP>:11434"
```

Apply with:
```bash
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

---

## OpenClaw Configuration (CT105)

| Setting | Value |
|---------|-------|
| Config file | `/root/.openclaw/openclaw.json` |
| Provider name | `archollama` |
| Base URL | `http://<ARCH-TAILSCALE-IP>:11434` |
| API adapter | `ollama` (native) |
| Default model | `archollama/qwen3.5:27b` |
| Gateway port | 18789 (local to CT105) |
| Gateway auth | Persistent token (stored in config) |

### Provider patch file (`/tmp/oc-patch.json`)

```json
{
  "models": {
    "providers": {
      "archollama": {
        "api": "ollama",
        "baseUrl": "http://<ARCH-TAILSCALE-IP>:11434",
        "models": [
          {
            "id": "qwen3.5:27b",
            "name": "Qwen 3.5 27B (Arch Ollama)",
            "input": ["text"],
            "contextWindow": 32768
          }
        ]
      }
    }
  }
}
```

Setup commands:
```bash
openclaw setup
openclaw config patch --file /tmp/oc-patch.json
openclaw models set archollama/qwen3.5:27b
openclaw models auth paste-api-key --provider archollama
openclaw config set gateway.auth.mode token
openclaw config set gateway.auth.token <your-generated-token>
```

---

## Systemd Service (CT105)

File: `/etc/systemd/system/openclaw.service`

```ini
[Unit]
Description=OpenClaw AI Gateway
After=network-online.target tailscaled.service
Wants=network-online.target

[Service]
Type=simple
User=root
EnvironmentFile=/root/.openclaw.env
ExecStart=/usr/bin/openclaw gateway run --force
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=openclaw

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now openclaw
```

---

## Persistence Checklist

- [x] CT105 `onboot=1` in Proxmox
- [x] `openclaw.service` enabled and started
- [x] `tailscaled.service` enabled (auto-starts with package)
- [x] Ollama service enabled on Arch
- [x] Ollama Tailscale override persists via drop-in config
- [x] Gateway auth token persisted in `openclaw.json`

---

## Useful Commands

```bash
# SSH into CT105
ssh root@<CT105-TAILSCALE-IP>

# Check OpenClaw status from Proxmox host
pct exec 105 -- systemctl status openclaw
pct exec 105 -- journalctl -u openclaw -n 30 --no-pager

# Verify Ollama is reachable from CT105
pct exec 105 -- curl -s http://<ARCH-TAILSCALE-IP>:11434/api/tags

# List agents / model status
pct exec 105 -- openclaw agents list
pct exec 105 -- openclaw models status

# Restart services after reboot
pct exec 105 -- systemctl restart tailscaled openclaw
```

---

## Adding More Ollama Models

```bash
# On the Arch machine
ollama pull <model-name>
```

Add the model to the provider's `models` array in the patch file, then:
```bash
openclaw models set archollama/<model-name>
systemctl restart openclaw
```

---

## Adding Chat Channels (Telegram, Discord, etc.)

```bash
pct exec 105 -- openclaw channels add
```

Follow the interactive prompts for your bot token/webhook.

---

## References

- OpenClaw docs: https://docs.openclaw.ai/cli
- Setup guide: https://jamesdonnelly.dev/blog/clawdbot-vps-setup/
- Tailscale in LXC: requires TUN device passthrough via cgroup2 + mount entry
