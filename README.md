# HomeLab

Personal homelab: self-hosted infrastructure, AI inference, and security research.

---

## Infrastructure

### Hardware

| Component | Spec |
|-----------|------|
| CPU | AMD Ryzen 5 5600X (6c/12t) |
| RAM | 32 GB DDR4 |
| Storage | 1 TB NVMe (TeamGroup G50) |
| GPUs | 2× NVIDIA GTX 1080 (VFIO passthrough) |
| PSU | 850 W fully modular |
| Form factor | ATX open-air test bench |

### Hypervisor: Proxmox VE 9.0

Runs headless at `192.168.1.100`. VMs live on `local-lvm` (LVM-Thin, 815 GiB). Web UI on `:8006`.

#### Active VMs

| VM | OS | Purpose |
|----|-----|---------|
| **Kali** (102) | Kali Linux | Bug hunting, recon, offensive security. Runs the Hermes Agent AI stack: a Telegram-driven, local-LLM recon and exploitation pipeline. |
| **AI-interface** (104) | Ubuntu Server 22.04 | Local AI inference via Ollama. Both GTX 1080s passed through via VFIO. |
| **TrueNAS** (105) | TrueNAS Scale 25.10 | NAS. ASMedia ASM1166 SATA controller passed through via VFIO; ZFS mirrors serving SMB/NFS and acting as the Proxmox backup target. |

### NAS (TrueNAS Scale, VM 105)

Built on 4 drives (2×3 TB + 2×1 TB) with the SATA controller passed through to the VM.

- **ZFS pools:** `family` (3 TB + 3 TB mirror, 2.7 TB usable) and `personal` (1 TB + 1 TB mirror, 928 GB usable).
- **Datasets:** per-member private family shares, a shared family folder, personal files, a Windows PC backup target, and an NFS dataset for Proxmox VM backups.
- **Shares:** SMB for Windows/Mac clients; NFS export restricted to the Proxmox host for nightly VM backups (zstd, keep-last 3).
- **Remote access:** reachable over Tailscale (`100.103.179.13`, hostname `truenas-scale`).

---

## Projects

### [Network Security Analyzer](https://github.com/MazenJ200/Network-Security-Analyzer)

Graduation-project-grade network packet analyzer.

- Live packet capture with anomaly detection
- Real-time web dashboard
- Docker support for easy deployment
- Built with Python, Scapy, and a Flask/WebSocket frontend

---

### [Kali AI Stack](https://github.com/MazenJ200/kali-ai-stack) *(Kali VM)*

An end-to-end automated security research pipeline running on the Kali VM, backed by local LLMs on the AI-interface VM and driven conversationally through [Hermes Agent](https://github.com/NousResearch/hermes-agent) over Telegram.

**Flow:**
```
Target domain
    ↓
recon_pipeline.sh: subfinder, httpx, naabu, nmap, gobuster, ffuf, dalfox, sqlmap, nuclei, ...
    ↓
ai_pipeline.py (3 passes on local Ollama):
    Ghost   (hermes3:8b)        → organize raw findings into JSON
    Oracle  (qwen2.5-coder:14b) → triage, CVSS, attack chains
    Exploit (deepseek-r1:14b)   → exploitation steps + payloads
    ↓
Report file + Telegram notification (Hermes Agent)
```

The Kali VM offloads all inference to the dual-GPU Ollama instance on the AI-interface VM over the OpenAI-compatible API. Authorized bug bounty and penetration-testing scope only. (This replaced an earlier OpenClaw-based orchestration layer.)

---

### Hermes Agent: AI Orchestration *(Kali VM)*

[Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research) is the agent layer on the Kali VM, replacing the previous OpenClaw setup. It runs as a systemd-user gateway service and provides:

- **Telegram control** — default-deny allowlist; send `hunt <target>` to launch the full pipeline and receive reports back in chat.
- **Local-first models** — Ollama on the AI-interface VM via the OpenAI-compatible endpoint, with cloud fallbacks optional.
- **Custom skills** — `run-hunt` (drives the pipeline), `mr-rob` (exploit-execution persona), and `godmode`.

---

### reconFTW: Full Recon Suite *(Kali VM)*

reconFTW is installed and fully configured on the Kali VM with all optional tools. Used for authorized full-scope recon against in-scope targets.

A full `-a` scan typically surfaces:
- Large subdomain sets across multiple independent attack surfaces
- Exposed services and mail infrastructure
- Dev/staging environments and legacy systems

A custom bash recon pipeline chains reconFTW → httpx → naabu → nmap into a single organized run, with output ready to feed directly into the AI analysis pipeline.

---

### Custom Recon Script *(Kali VM)*

Wraps the full toolchain into one command. Takes a domain, creates a structured output folder, and runs:

1. Subdomain enumeration via reconFTW
2. Live host filtering via httpx
3. Fast port scan via naabu (top 1000 ports)
4. Service version scan via nmap
5. Web tech fingerprinting via whatweb / webanalyze
6. Consolidates all output for AI ingestion

---

### Self-Hosted Email Server *(planned / in-progress)*

A Mailcow-based self-hosted mail server running on Ubuntu 24 in Proxmox. Goal is to host email for personal and family use on a custom domain.

Stack: Mailcow (Docker): Postfix + Dovecot + DKIM + DMARC + spam filtering. Custom domain ending in `.work`.

---

### OpenRA: Combined Arms AI *(local, Windows)*

Custom bot personality for OpenRA Red Alert (Combined Arms mod).

Merges the strongest traits of all four stock AIs (Rush / Normal / Turtle / Naval) into one aggressive, well-rounded opponent. Key tuning: faster radar intel (2000 cash threshold vs 3000), higher harvester cap (12), minelayer usage, earlier nuke/paratrooper thresholds.

*Not yet published to GitHub.*

---

## Network

| Host | IP | Notes |
|------|----|-------|
| Proxmox (Maz) | `192.168.1.100` | Hypervisor host. Tailscale: `100.115.132.15` |
| Kali VM | `192.168.1.187` | Tailscale: `100.127.159.57` |
| AI-interface VM | `192.168.1.171` | Ollama on `0.0.0.0:11434` (firewall: Kali only) |
| TrueNAS VM | `192.168.1.225` | Tailscale: `100.103.179.13` (`truenas-scale`) |
