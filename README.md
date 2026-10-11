# HomeLab

Personal homelab for self-hosted infrastructure, local AI inference, and hands-on security research: a two-node Proxmox setup running production-style services, a GPU-backed local LLM stack, and an on-demand offensive-security lab.

---

## Architecture

Two independent Proxmox VE nodes on a 2.5GbE backbone, joined over Tailscale for remote access and managed from a single Proxmox Datacenter Manager dashboard.

| Node | Role | GPUs |
|------|------|------|
| **Ryzen** (AMD Ryzen 5 5600X) | NAS + local AI inference | 3× GTX 1080 (24 GB, VFIO → AI VM) |
| **Dell Precision 5820** | Security lab + media + web edge | 1× GTX 1080 (host driver: Jellyfin NVENC, photo ML, console) |

Design note: the earlier "both boards are 2-GPU-max" conclusion was a misdiagnosis. A third card knocked the Ryzen's NIC off the bus because PCI address space below 4 GB ran out (MMIO/BAR exhaustion), not because of lane or power limits. Enabling Above 4G Decoding fixed it, and swapping the M.2 devices cleared the blocked x16 slot while moving the SATA controller onto CPU lanes, so all three 1080s now pool to 24 GB for inference and the 5820's remaining card serves media.

---

## Node 1: Ryzen (NAS + AI)

### Hardware

| Component | Spec |
|-----------|------|
| CPU | AMD Ryzen 5 5600X (6c/12t) |
| RAM | 32 GB DDR4 |
| Boot | 1 TB NVMe |
| GPUs | 3× NVIDIA GTX 1080 (VFIO passthrough; one in the CPU x16 slot, two on chipset x1 risers), capped at 150 W each |
| Storage controller | ASM1166 6-port SATA on a CPU-attached M.2 slot (VFIO → TrueNAS) |
| Network | 2.5GbE add-in NIC for the bridge; onboard 1GbE kept off the bridge as a rescue port |
| PSU | 850 W fully modular |
| Chassis | ATX open-air bench |

Hypervisor: Proxmox VE 9. Both NICs are pinned by MAC (systemd `.link`) so adding or moving a GPU can never rename the bridge interface and knock the host off the network, and every passthrough device is pinned by ID and IOMMU group through Proxmox resource mappings, so a hardware change fails safe instead of silently handing a VM the wrong device.

### Guests

| Guest | Purpose |
|-------|---------|
| **AI inference** (VM) | Ubuntu 24.04 + Ollama, all three 1080s passed through and pooled to 24 GB. Serves a local, OpenAI-compatible endpoint, firewalled so only the Kali VM and the ops agent can reach it. A ~20 GB quantized model stays fully resident in VRAM, split across all three cards. |
| **TrueNAS SCALE** (VM) | ZFS NAS. ASMedia SATA controller passed through via VFIO; `family` and `personal` mirror pools serve SMB/NFS, hold the photo and media libraries, and act as the Proxmox backup target. |
| **Ops agent** (LXC) | Read-only Hermes agent for checking on the lab from Telegram (see Projects). |

### NAS

- ZFS mirrors: ~2.7 TB usable `family` pool (2× 3 TB) and ~0.9 TB usable `personal` pool (2× 1 TB).
- Per-member private shares, a shared family folder, a Windows PC backup target, the media library, and an NFS dataset for Proxmox VM backups.
- SMB for Windows/Mac clients; the Proxmox backup export is restricted to the two Proxmox hosts. Reachable over Tailscale.
- Personal files and PC backups get daily and weekly snapshots plus encrypted offsite replication.

### Photos (Immich)

A self-hosted photo platform for the family, running as a TrueNAS app with its machine-learning worker offloaded to the 5820's GPU.

- Three users on ZFS mirrors, with hourly, daily, and weekly snapshots and nightly database dumps copied across pools.
- Encrypted offsite replication (rclone crypt) to object storage.
- Restore verified in September 2026: 15,123 of 15,123 files, with database dumps and sampled originals byte-identical by SHA-256.
- The ML worker (CLIP search, face recognition) is pinned to the GPU by UUID with a short model TTL, so indexing never starves Jellyfin transcodes on the same card.

---

## Node 2: Dell 5820 (Security lab + media + edge)

Dell Precision 5820 (Intel Core i9-10900X, 10c/20t, 32 GB) running Proxmox VE 9. Its remaining GTX 1080 stays on the host NVIDIA driver rather than VFIO: it is shared into the Jellyfin and Immich ML containers for NVENC and CUDA, and doubles as the physical console (learned the hard way that an all-VFIO headless host plus a renaming NIC equals a full lockout). When a reboot renumbered the GPU's device node and silently broke hardware transcoding, the containers moved to Proxmox device entries, and GPU checks now run real encode work instead of trusting `nvidia-smi`.

| Guest | Purpose |
|-------|---------|
| **Kali** (VM) | Bug-hunting / offensive-security workstation, powered on only for engagements. No GPU since its 1080 joined the inference pool; hosts the AI recon stack below, with inference on the Ryzen. |
| **Web** (VM) | Debian + Docker host for two production websites, each in its own container, published through a Cloudflare Tunnel. |
| **Jellyfin** (LXC) | Media stack with NVENC hardware transcoding on the host GPU. |
| **Immich ML** (LXC) | Photo machine-learning worker (CLIP search, face recognition) on the host GPU. |
| **PDM** (LXC) | Proxmox Datacenter Manager, one pane of glass across both nodes. |
| **Monitoring** (LXC) | Beszel + Uptime Kuma with Telegram alerting across every node and service. |
| **Apps** (LXC) | Homepage dashboard, What's Up Docker, and the manual update console. |

---

## Projects

### [Kali AI Stack](https://github.com/MazenJ200/kali-ai-stack)

A local, AI-driven recon and triage pipeline for **authorized** security testing. A Telegram-driven agent (Hermes Agent over local Ollama) takes a target and runs recon, structuring, triage/CVSS scoring, and an exploitation-advisory pass, ending in a report. All inference runs on the AI node, so no target data leaves the lab.

### Read-Only Ops Agent

A local LLM agent that answers lab questions from my phone over Telegram (GPU load, disk usage, service health) and is deliberately unable to change anything. Three layers keep it that way: a per-platform tool allowlist with no shell toolset, a custom nine-tool read-only MCP server, and an SSH forced-command dispatcher that accepts only fixed verbs and rate-limits calls. A self-adversarial design review caught an exploitable path before deployment, and the residual risk is written down rather than hand-waved.

### [Network Security Analyzer](https://github.com/MazenJ200/Network-Security-Analyzer)

Graduation-grade network packet analyzer: live capture with anomaly detection, a real-time web dashboard, and Docker deployment. Built with Python, Scapy, and a Flask/WebSocket frontend.

---

## Operations

- **Remote access:** Tailscale mesh across both nodes and key guests; admin web UIs are additionally fronted by a Cloudflare Tunnel behind an email-gated Cloudflare Access policy, with no ports forwarded on the router.
- **Web hosting:** two production sites served from the lab through a Cloudflare Tunnel with automatic TLS and no inbound ports. The portfolio site sends HSTS, a content security policy, and the standard hardening headers.
- **Network:** 2.5GbE backbone, proven against a pre-registered iperf3 baseline: node-to-node throughput up from ~936 to 2360 Mbps, NFS writes up 91%, and thousands of retransmits per run down to zero. A link monitor alerts if any NIC silently renegotiates below its expected speed.
- **Power:** the Ryzen node and core network gear sit on a UPS managed by NUT with a staged shutdown. On battery, the AI VM stops at +20 seconds to shed GPU load, the host follows gracefully at +5 minutes, and the AI VM restarts itself if power returns first. Per-card power caps keep draw predictable, and a small NVML daemon enforces a fan floor on the fan-stop cards.
- **Backups:** Two staggered, bandwidth-throttled weekly Proxmox jobs to the NAS (zstd, fleecing, keep-last pruning). The ZFS pools are passed through to TrueNAS, so they sit outside those jobs; the photo library, personal files, and PC backups are covered by snapshots and encrypted offsite replication instead.
- **Updates:** What's Up Docker checks every container on both nodes daily, and a Saturday Telegram digest covers host packages, guests, and apps. Nothing updates itself except OS security patches; a console shows what is behind and applies an update only when I press the button, tagging the old image for rollback. Major versions wait for a planned upgrade with a backup first.
- **Monitoring:** Beszel agents plus Uptime Kuma probes on every node and service, with Proxmox, TrueNAS, UPS, and link alerts all landing in Telegram.
- **Resilience:** MAC-pinned NICs, passthrough devices pinned by resource mapping, static guest addressing, a rescue NIC on the Ryzen, and a host console on the 5820.

---

## Toolchain (selected)

The security lab runs a broad recon and testing toolchain; a representative subset includes subfinder, httpx, nmap, nuclei, and ffuf, layered under a local-LLM analysis stage on Ollama. Additional tooling and pipeline internals are intentionally left out of this overview.
