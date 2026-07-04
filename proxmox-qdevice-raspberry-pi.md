# Proxmox QDevice Setup on Raspberry Pi
> Configure a Raspberry Pi as a Proxmox quorum device (QDevice) to enable High Availability on a 2-node cluster

---

## What is a QDevice?

In a 2-node Proxmox cluster, neither node can win a vote alone during a split-brain scenario.
A **QDevice** is a lightweight tiebreaker — it adds a third vote without needing a full Proxmox node.
The Raspberry Pi is perfect for this role: low power draw, always-on, cheap.

```
Node 1 (T1700)   →  1 vote
Node 2 (NAS)     →  1 vote
QDevice (RPi)    →  1 vote  ← tiebreaker
```

---

## Requirements

| Item | Details |
|---|---|
| Raspberry Pi | Any model with network access |
| OS | Raspberry Pi OS (Debian-based) |
| Network | Same network as Proxmox nodes |
| Proxmox cluster | Already created with at least 2 nodes |

---

## Step 1 — Prepare the Raspberry Pi

### Update the system
```bash
sudo apt update && sudo apt upgrade -y
```

### Install corosync-qnetd
```bash
sudo apt install corosync-qnetd -y
```

### Enable and start the service
```bash
sudo systemctl enable corosync-qnetd
sudo systemctl start corosync-qnetd
```

### Verify it is running
```bash
sudo systemctl status corosync-qnetd
```

> You should see `active (running)`. Note the Raspberry Pi IP address — you will need it.

---

## Step 2 — Install corosync-qdevice on Proxmox Nodes

Run this on **every Proxmox node** in the cluster:

```bash
apt update
apt install corosync-qdevice -y
```

---

## Step 3 — Add the QDevice to the Cluster

Run this on **Node 1 (T1700)** only:

```bash
pvecm qdevice setup <raspberry-pi-ip>
```

Example:
```bash
pvecm qdevice setup 192.168.1.50
```

> This will SSH into the Raspberry Pi and configure the QDevice automatically.
> Make sure SSH is enabled on the Raspberry Pi.

---

## Step 4 — Verify the QDevice

```bash
pvecm status
```

You should see output like:

```
Quorum information
------------------
Date:             ...
Quorum provider:  corosync_votequorum
Nodes:            2
Node votes:       2
Quorum votes:     2
Expected votes:   3
Total votes:      3
Quorum:           2
Flags:            Quorate

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2
Flags:            Quorate

Membership information
----------------------
Nodeid  Votes  Qdevice  Name
     1      1    A,V,NMW node1 (local)
     2      1    A,V,NMW node2
     0      1            Qdevice
```

> `Qdevice` should appear with 1 vote. Total = 3 votes. Cluster is quorate. ✅

---

## Step 5 — Enable HA on the Cluster

Once the QDevice is active, enable HA in the Proxmox web UI:

1. Go to **Datacenter > HA**
2. Click **Add** under Resources
3. Select your container or VM
4. Set **State** to `started`
5. Set **Max Restart** = 3
6. Set **Max Relocate** = 2
7. Click **Add**

---

## Troubleshooting

### QDevice not connecting
```bash
# Check SSH access from Proxmox node to Raspberry Pi
ssh root@<raspberry-pi-ip>

# Check qnetd status on Raspberry Pi
sudo systemctl status corosync-qnetd

# Check qdevice status on Proxmox node
systemctl status corosync-qdevice
```

### Reset QDevice
```bash
# On Proxmox Node 1
pvecm qdevice remove

# Then re-run setup
pvecm qdevice setup <raspberry-pi-ip>
```

---

## Architecture Summary

```
192.168.1.108   →  Proxmox Node 1 (T1700)    port 8006
192.168.1.x     →  Proxmox Node 2 (NAS)      port 8006
192.168.1.50    →  QDevice (Raspberry Pi)     corosync-qnetd
```

---

## Power Draw Comparison

| Device | Idle Wattage |
|---|---|
| Dell T1700 | ~40W |
| Raspberry Pi 4 | ~3W |
| NAS (future) | ~15-30W |

> The Raspberry Pi adds almost no cost to run 24/7 as a QDevice.

---

## Author

**Dinaverse** — [GitHub](https://github.com/Dinaverse) | [LinkedIn](https://www.linkedin.com/in/dinacima)
