# Incident Report & Network Architecture Guide

**Author:** Dina Cima Mufungizi  
**Date:** July 5, 2026  
**Environment:** Proxmox VE, Tailscale, OPNsense & Kali Linux

---

## 1. Lab Network Architecture Overview

This document summarises the resolution of critical routing conflicts encountered during the deployment of the virtual security infrastructure.

| Equipment / VM | Physical / Virtual Interface | IPv4 LAN Address | Role in Infrastructure |
|---|---|---|---|
| Tenda Router | Physical (Main Gateway) | `<gateway-ip>` | Home internet gateway / Base DHCP server |
| Proxmox Host (`dina`) | `vmbr0` (Physical Bridge) | `<proxmox-ip>` | Main hypervisor and Tailscale subnet router |
| Kali Linux Machine | `eth0` (Physical) | `<kali-ip>` | Admin and audit operator workstation |
| OPNsense (VM 200) – WAN | `vtnet0` (bridged on `vmbr0`) | `<opnsense-wan-ip>` (DHCP) | External interface of the lab firewall |
| OPNsense (VM 200) – LAN | `vtnet1` (bridged on `vmbr1`) | `<opnsense-lan-ip>` | Gateway of the isolated lab subnet |

---

## 2. Incident #1 — LAN Blackout via Tailscale Routing Conflict

### Symptom: Total Loss of Local Access (HTTP & ICMP Timeout)

Unable to `curl` or `ping` the main router (`<gateway-ip>`) or the OPNsense interface (`<opnsense-wan-ip>`) from the Kali Linux machine, despite active internet connectivity. All requests returned error code `(28) Failed to connect`.

### Root Cause: Subnet Routing Collision

The **"Accept Subnet Routes"** option was simultaneously active in the Tailscale admin panel for multiple devices (`dina`, `kali`, `pi-hole`). All machines were advertising ownership of the physical local subnet `192.168.1.0/24`. The Kali Linux kernel was prioritising the encrypted virtual `tailscale0` interface over the physical `eth0` interface, routing all local LAN traffic into a virtual black hole.

### Resolution & Stable Configuration

To isolate local network traffic while maintaining encrypted remote SSH access, the following procedure was applied:

**Step 1 — Temporarily stop the daemon and flush neighbour tables on Kali:**

```bash
sudo tailscale down
sudo ip neigh flush all
```

**Step 2 — Permanently disable subnet route sharing for the `192.168.1.0/24` network on the Kali machine only**, via the Tailscale web admin console (Route Settings).

**Step 3 — Restart the Tailscale agent on Kali with an explicit prohibition on intercepting physical LAN routes:**

```bash
sudo tailscale up --accept-routes=false --snat-subnet-routes=false
```

---

## 3. Incident #2 — Internet Loss and DNS Failure on Proxmox

### Symptom: Apt Repository Resolution Failure (`Temporary failure resolving`)

The Proxmox host became unable to update its packages (`download.proxmox.com`) and showed 100% packet loss during ICMP requests to external public IPs such as `1.1.1.1`.

### Root Cause: OPNsense Default Deny Rule & Corrupted Route

Activating the OPNsense WAN interface on the same bridge (`vmbr0`) triggered OPNsense's default security filtering, blocking outbound network traffic from the hypervisor itself. In parallel, the Proxmox routing table contained an unstable gateway directive of type `proto kernel onlink`.

### Resolution & Stable Configuration

**Step 1 — Temporarily disable the OPNsense filter engine from the console to regain control:**

```bash
pfctl -d
```

**Step 2 — Correct and stabilise the routing table on the Proxmox host (`dina`):**

```bash
ip route del default
ip route add default via <gateway-ip> dev vmbr0 proto static
ip neigh flush dev vmbr0
```

**Step 3 — Force a robust, functional DNS configuration in the system file:**

```bash
echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" > /etc/resolv.conf
```

---

## 4. Permanent OPNsense Configuration (Double-NAT & Lab Access)

To maintain access and prevent OPNsense from blocking the Proxmox hypervisor or the Kali operator again when firewall rules are active, the following critical adjustments were applied via the OPNsense GUI (`https://<opnsense-wan-ip>`):

### 4.1 Disable Private/Bogon Network Blocking on WAN

Since the OPNsense WAN interface sits behind a domestic Tenda router (Double-NAT environment), it is necessary to disable the default internet protection standards:

1. Navigate to: **Interfaces → [WAN]** (or identifier `opt1` associated with `vtnet0`)
2. Uncheck: **Block private networks**
3. Uncheck: **Block bogon networks**
4. Save and apply changes.

### 4.2 Create Firewall Bypass Rules

Two explicit **Pass** rules were inserted at the top of the WAN (`opt1`) filter matrix to protect access for admin machines:

| Action | Interface | Protocol | Source | Destination | Description |
|---|---|---|---|---|---|
| PASS | WAN (`opt1`) | any | `<proxmox-ip>` (Proxmox) | any | Bypass Proxmox Host Traffic |
| PASS | WAN (`opt1`) | TCP/HTTPS | `<kali-ip>` (Kali Linux) | WAN address (port 443) | Bypass Kali GUI Management |

### Golden Rule for Adding Future Clients

Any new physical machine that needs to administer the lab from the local home network must have a static IP (or a reserved DHCP lease) and must have a similar **Pass** rule created on the OPNsense WAN interface.

For remote connections from outside the network, the device must use the Tailscale agent with the **Accept Subnet Routes** option enabled to transit in encrypted form through the approved Proxmox host.

---

*Documentation Technique Lab — Dina Cima Mufungizi*
