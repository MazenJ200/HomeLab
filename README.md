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
| **AI-interface** (104) | Ubuntu Server 22.04 | Local AI inference via Ollama. Both GTX 1080s passed through via VFIO. |

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

### AI-Powered Vulnerability Pipeline *(Kali VM)*

An end-to-end automated security research pipeline running on the Kali VM, backed by local LLMs on the AI-interface VM.

**Flow:**
```
Target domain
    ↓
reconFTW + supplemental tools (nuclei, katana, ffuf, sqlmap, dalfox, httpx, naabu, nikto, etc.)
    ↓
Aggregator script → structured prompt-ready output
    ↓
Ollama "Organizer" (qwen3.5:27b) → normalized findings
    ↓
Ollama "Analyzer" (deepseek-coder-v2:16b) → vuln triage + severity ratings
    ↓
Report file + Telegram notification via OpenClaw
```

The Kali VM connects to the AI-interface VM over SSH to offload inference to the dual-GPU Ollama instance. The pipeline targets authorized domains for bug bounty and authorized penetration testing.

---

### OpenClaw — AI Agent Setup *(Kali VM)*

[OpenClaw](https://openclaw.ai) is installed on the Kali VM as the agent orchestration layer. Configured with:

- **ClawBot** — Telegram bot integration for receiving scan notifications and interacting with agents remotely
- **Multiple LLM backends** — local Ollama models on the AI-interface VM + cloud fallbacks
- **Pentesting agent** — custom agent wired to run recon tools, parse output, and deliver structured vulnerability reports through the pipeline above

---

### reconFTW — Full Recon Suite *(Kali VM)*

reconFTW is installed and fully configured on the Kali VM with all optional tools. Used for authorized full-scope recon against targets, including `meu.edu.jo` (Middle East University — authorized scope).

A full scan against `meu.edu.jo -a` surfaced:
- 100+ subdomains across multiple independent attack surfaces
- Exposed mail server (SMTP/POP3/IMAP/FTP all open on `beds.meu.edu.jo`)
- Dev/staging environments and legacy systems

Also written: a custom bash recon pipeline script that chains reconFTW → httpx → naabu → nmap into a single organized run, with output files ready to feed directly into the AI analysis pipeline.

---

### Custom Recon Script *(Kali VM)*

Wraps the full toolchain into one command. Takes a domain, creates a structured output folder, and runs:

1. Subdomain enumeration via reconFTW
2. Live host filtering via httpx
3. Fast port scan via naabu (top 1000 ports)
4. Service version scan via nmap
5. Web tech fingerprinting via whatweb / webanalyze
6. Consolidates all output into `all_scans.txt` for AI ingestion

---

### Self-Hosted Email Server *(planned / in-progress)*

A Mailcow-based self-hosted mail server running on Ubuntu 24 in Proxmox. Goal is to host email for personal use and family on a custom domain.

Stack: Mailcow (Docker) — Postfix + Dovecot + DKIM + DMARC + spam filtering, all pre-configured. Custom domain ending in `.work`.

---

### OpenRA — Combined Arms AI *(local, Windows)*

Custom bot personality for OpenRA Red Alert (Combined Arms mod).

Merges the strongest traits of all four stock AIs (Rush / Normal / Turtle / Naval) into one aggressive, well-rounded opponent. Key tuning: faster radar intel (2000 cash threshold vs 3000), higher harvester cap (12), minelayer usage, earlier nuke/paratrooper thresholds.

*Not yet published to GitHub.*

---

## Network

| Host | IP | Notes |
|------|----|-------|
| Proxmox (Maz) | `192.168.1.100` | Hypervisor host |
| Kali VM | `192.168.1.187` | Tailscale: `100.127.159.57` |
| AI-interface VM | `192.168.1.171` | Ollama on `0.0.0.0:11434` (firewall: Kali only) |
