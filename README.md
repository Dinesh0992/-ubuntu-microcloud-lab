# Ubuntu MicroCloud Learning Lab

**Status:** Architecture Pivot Complete → Single-Node LXD Hypervisor Operational  
**Last Updated:** June 16, 2026  
**Hardware:** AMD FX-8350 Bare-Metal Server  
**Network:** `enp3s0` @ `192.168.1.7` / `192.168.1.8`  
**Orchestration:** LXD (Non-Clustered) + HTTPS Dashboard (`:8443`)

---

## Project Overview

This repository documents a home lab learning journey for **microservice distributed systems and Ubuntu MicroCloud VM creation** on Ubuntu. The project began with a MicroCloud multi-node cluster architecture but pivoted to a lean **single-node LXD hypervisor** after hardware failures and architectural analysis revealed MicroCloud is over-engineered for single-node learning workloads.

### Key Architectural Decisions

| Date | Decision | Rationale |
|------|----------|-----------|
| **Jun 12** | Purged `microcloud`, `microceph`, `microovn` snaps | Multi-node cluster software caused `dqlite` corruption, AppArmor loops, CPU overhead on single node |
| **Jun 12** | Migrated SSD from SATA III → SATA II (Port 4) | Legacy AMD SB950 chipset timing desync with NCQ caused `ata5.00` freezes |
| **Jun 16** | Injected missing `admins` RBAC group into LXD | LXD build lacked pre-configured admin group, blocking TLS trust token validation |
| **Jun 16** | Deployed `lab-node1` (Ubuntu 24.04) | Verified LXD engine provisioning mechanics end-to-end |

---

## Timeline & Milestones

### June 12, 2026 — Hardware Stabilization & Architecture Pivot
- **Root Cause:** `ata5.00: exception Emask 0x10... frozen` kernel errors during `microcloud init`
- **Fix:** Physical SATA cable migration from Port 5 (SATA III) → Port 4 (SATA II)
- **Validation:** `smartctl` → `PASSED`, zero `Emask`/`frozen` errors in `journalctl -k`
- **Pivot:** Removed MicroCloud stack (`snap remove --purge microcloud microceph microovn`), reverted to raw LXD
- **Baseline:** Clean LXD state, network active at `192.168.1.8`, SSD healthy

### June 16, 2026 — LXD TLS Identity, RBAC & Dashboard Access
- **Network Debug:** Fixed dead Ethernet link/ARP failure between server and Airtel gateway
- **SSH Hardening:** Verified `ED25519` host key acceptance
- **LXD RBAC Fix:** Injected missing security schema:
  ```bash
  sudo lxc auth group create admins
  sudo lxc auth group permission add admins server admin
  lxc auth identity create tls/lxd-ui --group admins
  ```
- **UI Access:** Browser authenticated instantly at `https://192.168.1.7:8443` (dark mode)
- **Workload Test:** `sudo lxc launch ubuntu:24.04 lab-node1` → RUNNING, IP acquired, visible in dashboard

---

## Current System State

```
┌─────────────────────────────────────────────────────────────┐
│                    AMD FX-8350 Server                       │
├─────────────────────────────────────────────────────────────┤
│  Hardware:     SSD @ SATA Port 4 (SATA II) — SMART: PASSED  │
│  Network:      enp3s0 → 192.168.1.7/8                       │
│  Hypervisor:   LXD 5.x (non-clustered, snap)                │
│  Dashboard:    https://192.168.1.7:8443 (TLS client cert)   │
│  Auth:         admins group + tls/lxd-ui identity ✓         │
│  Containers:   lab-node1 (ubuntu:24.04) — RUNNING           │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Reference: Operational Commands

### LXD Administration
```bash
# Initialize LXD (first time only)
sudo lxd init

# RBAC: Create admin group & grant permissions
sudo lxc auth group create admins
sudo lxc auth group permission add admins server admin

# Generate trust token for browser UI
lxc auth identity create tls/lxd-ui --group admins
# → Copy the printed token into https://<ip>:8443
```

### Container Lifecycle
```bash
# Launch workload
sudo lxc launch ubuntu:24.04 <name>

# Execute into container
sudo lxc exec <name> bash

# List all instances
sudo lxc list

# Stop all containers gracefully
sudo lxc stop --all
```

### Network Diagnostics
```bash
# Check IP assignments
hostname -I
ip link show
ip addr show enp3s0

# Verify ARP/gateway reachability
arp -a
ping -c 3 192.168.1.1
```

### Graceful Lab Teardown (Runbook)
```bash
# 1. Halt all workloads (unmount FS, stop daemons)
sudo lxc stop --all

# 2. Verify zero active instances
sudo lxc list

# 3. Power down physical host
sudo shutdown -h now
```

---

## Learning Roadmap

### Phase 1: Foundation (COMPLETE ✓)
- [x] Hardware stabilization (SATA migration)
- [x] MicroCloud → LXD pivot
- [x] LXD TLS/RBAC configuration
- [x] Dashboard access verified
- [x] Base container deployment verified

### Phase 2: Microservice Infrastructure (NEXT)
- [ ] Deploy microservice containers (API gateway, services, databases)
- [ ] Configure service discovery & load balancing
- [ ] Implement inter-service communication (gRPC/REST)

### Phase 3: Distributed Systems Patterns
- [ ] Design distributed transactions & eventual consistency
- [ ] Implement circuit breakers, retries, timeouts
- [ ] Test failure scenarios & resilience patterns

### Phase 4: Observability & Resilience
- [ ] Add Prometheus/Grafana stack
- [ ] Configure structured logging
- [ ] Chaos testing (container kill, network partition)

---

## Reference Documents

| File | Description |
|------|-------------|
| `Consolidated_Engineering_Log_2026-06-12.md` | End-of-day summary: hardware fix, snap purge, LXD pivot |
| `Full_Day_Analysis_2026-06-12.md` | Detailed RCA: `ata5.00` freezes, dqlite corruption, AppArmor loops |
| `Simplifying your distributed cloud infrastructure with MicroCloud.md` | Canonical whitepaper (Q4 2024) — MicroCloud architecture & deployment guide |

---

## Dashboard Screenshots

### Initial LXD UI State
![LXD UI Initial State](Images/LxdUIStartInitial.png)

### LXD UI with lab-node1 Running
![LXD UI with lab-node1](Images/LxDUIWithUbuntulab-node1.png)

---

## Why Not MicroCloud?

> **MicroCloud is designed for multi-node HA clusters** (3+ nodes for Ceph quorum).  
> Our single-node lab gained only: `dqlite` corruption, AppArmor denial loops, CPU overhead from orchestration daemons.  
> **Raw LXD** gives direct control over containers, storage, and networking — ideal for learning microservice distributed systems and VM creation without distributed systems complexity.

> *From the whitepaper: "MicroCloud supports single-member deployments which are great for testing... For HA, and the ability to recover a cluster should something go wrong, you need a minimum of three members."*

---

## License & Attribution

- **Project Docs:** Personal learning lab — no license
- **MicroCloud Whitepaper:** © 2024 Canonical Limited (included for reference)
- **Ubuntu/LXD:** Canonical trademarks

---

*Generated from engineering logs — keep this README updated as the lab evolves.*