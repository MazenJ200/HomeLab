# HomeLab

Personal homelab — self-hosted infrastructure, AI inference, and security research.

---

## Infrastructure

### Hardware

| Component | Spec |
|-----------|------|
| CPU | AMD Ryzen 5 5600X (6c/12t) |
| RAM | 32 GB DDR4 |
| Storage | 1 TB NVMe (TeamGroup G50) |
| GPUs | 2× NVIDIA GTX 1080 (VFIO passthrough) |
| PSU | 650 W fully modular (850 W upgrade staged) |
| Form factor | ATX open-air test bench |

### Hypervisor — Proxmox VE 9.0

Runs headless at `192.168.1.100`. All VMs live on `local-lvm` (LVM-Thin, 815 GiB). Web UI on `:8006`.

#### Active VMs

| VM | OS | Purpose |
|----|-----|---------|
| **Kali** (102) | Kali Linux | Bug hunting, recon, offensive security. Runs OpenClaw with AI agent integration. |
| **AI-interface** (104) | Ubuntu Server 22.04 | Local AI inference via Ollama. Both GTX 1080s passed through via VFIO. Running DeepSeek-R1-Distill-Qwen-14B (Q5_K_M). |

### Planned — NAS

TrueNAS build with 4 drives (2×3 TB + 2×1 TB = 8 TB raw). Hardware acquired; build pending.

---

## Projects

### [Network Security Analyzer](https://github.com/MazenJ200/Network-Security-Analyzer)

Graduation-project-grade network packet analyzer.

- Live packet capture with anomaly detection
- Real-time web dashboard
- Docker support for easy deployment
- Built with Python, Scapy, and a Flask/WebSocket frontend

---

### OpenRA — Combined Arms AI *(local)*

Custom bot personality for OpenRA Red Alert (Combined Arms mod).

Merges the strongest traits of all four stock AIs (Rush / Normal / Turtle / Naval) into one aggressive, well-rounded opponent. Key improvements: faster radar intel, higher harvester cap (12), earlier nuke/paratrooper thresholds, and minelayer usage.

*Not yet published to GitHub.*

---

## Network

| Host | IP | Notes |
|------|----|-------|
| Proxmox (Maz) | `192.168.1.100` | Hypervisor host |
| Kali VM | `192.168.1.187` | Tailscale: `100.127.159.57` |
| AI-interface VM | `192.168.1.171` | Ollama on `0.0.0.0:11434` (firewall: Kali only) |
