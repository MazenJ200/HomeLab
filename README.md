# HomeLab

Personal homelab for self-hosted infrastructure, local AI inference, and hands-on security research: a two-node Proxmox setup running production-style services, a GPU-backed local LLM stack, and an isolated offensive-security lab.

---

## Architecture

Two independent Proxmox VE nodes on a flat LAN, joined over Tailscale for remote access and managed from a single Proxmox Datacenter Manager dashboard.

| Node | Role | GPUs |
|------|------|------|
| **"Maz"** (AMD Ryzen) | NAS + local AI inference | 2× GTX 1080 (16 GB, VFIO → AI VM) |
| **"pve"** (Dell Precision 5820) | Offensive-security lab + edge services | 1× GTX 1080 (VFIO → Kali) + NVS 310 (host console) |

Design note: both boards turned out to be 2-GPU-max in practice (PCIe lane budget and power-connector limits), so inference stays at 16 GB on the Ryzen and the third 1080 drives GPU compute on the security node rather than forcing a single 24 GB pool.

---

## Node 1 — "Maz" (NAS + AI)

### Hardware

| Component | Spec |
|-----------|------|
| CPU | AMD Ryzen 5 5600X (6c/12t) |
| RAM | 32 GB DDR4 |
| Boot | 1 TB NVMe |
| GPUs | 2× NVIDIA GTX 1080 (VFIO passthrough) |
| PSU | 850 W fully modular |
| Chassis | ATX open-air bench |

Hypervisor: Proxmox VE 9. The onboard NIC is pinned by MAC (systemd `.link`) so adding or moving a GPU can never rename the bridge interface and knock the host off the network.

### Guests

| Guest | Purpose |
|-------|---------|
| **AI-Interface** (VM) | Ubuntu 24.04 + Ollama, both 1080s passed through (16 GB). Serves a local, OpenAI-compatible LLM endpoint to the rest of the lab. |
| **TrueNAS Scale** (VM) | ZFS NAS. ASMedia SATA controller passed through via VFIO; `family` and `personal` mirror pools serve SMB/NFS and act as the Proxmox backup target. |

### NAS

- ZFS mirrors: ~2.7 TB usable `family` pool (2× 3 TB) and ~0.9 TB usable `personal` pool (2× 1 TB).
- Per-member private shares, a shared family folder, a Windows PC backup target, and an NFS dataset for Proxmox VM backups.
- SMB for Windows/Mac clients; the NFS export is restricted to the two Proxmox hosts. Reachable over Tailscale.

---

## Node 2 — "pve" (Security lab + edge)

Dell Precision 5820 running Proxmox VE 9. VFIO binds one GTX 1080 to the Kali VM while an NVS 310 keeps a physical host console (learned the hard way that an all-VFIO headless host plus a renaming NIC equals a full lockout).

| Guest | Purpose |
|-------|---------|
| **Kali** (VM) | Bug-hunting / offensive-security workstation. GTX 1080 passthrough with CUDA/OpenCL for GPU-accelerated password cracking (hashcat, ~25 GH/s MD5). Hosts the AI recon stack below. |
| **PDM** (LXC) | Proxmox Datacenter Manager, one pane of glass across both nodes. |
| **Monitoring** (LXC) | Beszel + Uptime Kuma with Telegram alerting across every node and service. |
| **Smart Skies** (VM) | Staged public-application host, kept offline pending a proper DMZ. |

---

## Projects

### [Kali AI Stack](https://github.com/MazenJ200/kali-ai-stack)

A local, AI-driven recon and triage pipeline for **authorized** security testing. A Telegram-driven agent (Hermes Agent over local Ollama) takes a target and runs recon, structuring, triage/CVSS scoring, and an exploitation-advisory pass, ending in a report. All inference runs on the AI node, so no target data leaves the lab.

### [Network Security Analyzer](https://github.com/MazenJ200/Network-Security-Analyzer)

Graduation-grade network packet analyzer: live capture with anomaly detection, a real-time web dashboard, and Docker deployment. Built with Python, Scapy, and a Flask/WebSocket frontend.

---

## Operations

- **Remote access:** Tailscale mesh across both nodes and key guests; web UIs are additionally fronted by Cloudflare Tunnels behind an email-gated Cloudflare Access policy, with no ports forwarded on the router.
- **Backups:** Two staggered, bandwidth-throttled Proxmox jobs to the NAS (zstd, fleecing, keep-last pruning). The NAS VM itself is intentionally excluded to avoid a circular backup-to-self.
- **Monitoring:** Beszel agents plus Uptime Kuma probes on every node and service, with Telegram alerts.
- **Resilience:** MAC-pinned NICs, static guest addressing, and a retained host console on the GPU node.

---

## Toolchain (selected)

The security lab runs a broad recon and testing toolchain; a representative subset includes subfinder, httpx, nmap, nuclei, and ffuf, layered under a local-LLM analysis stage on Ollama. Additional tooling and pipeline internals are intentionally left out of this overview.
