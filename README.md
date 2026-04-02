# Self-Hosted Cloud Platform

A production-grade private cloud running on a home server — covering Docker orchestration, network security, VPN access, workflow automation, and infrastructure-as-code with Ansible.

---

## What I Built

A fully self-hosted alternative to Google Photos, Google Drive, and Zapier, running on a single home server with enterprise-grade security practices: encrypted remote access, layered firewalling, automated backups, and real-time monitoring.

**Hardware:** Intel Core i3-3220 @ 3.30 GHz · 8 GB RAM · Ubuntu 24.04.3 LTS (headless)

---

## Tech Stack

| Layer | Technologies |
|---|---|
| **Containerization** | Docker, Docker Compose |
| **Services** | Immich, Nextcloud, N8N, Cockpit |
| **Databases** | PostgreSQL, Redis, SQLite |
| **Networking** | Cloudflare (DNS · Proxy · SSL), Tailscale (WireGuard VPN) |
| **Security** | UFW, Fail2Ban, 2FA (TOTP), SSH key-only auth |
| **Automation** | N8N workflows, Telegram bot (Python + systemd), cron |
| **Backups** | restic / duplicity with GPG encryption |
| **IaC** | Ansible (see [ansible-homelab](https://github.com/iamthecloverly/ansible-homelab)) |

---

## Architecture

```
 ┌──────────────────────────────────────────────────────────┐
 │               Ubuntu 24.04 LTS Home Server               │
 │                                                          │
 │   ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
 │   │   Immich    │  │  Nextcloud  │  │      N8N       │  │
 │   │ (photos)    │  │ (file sync) │  │  (automation)  │  │
 │   └──────┬──────┘  └──────┬──────┘  └───────┬────────┘  │
 │          └────────────────┴──────────────────┘           │
 │                           │                              │
 │              ┌────────────┴───────────┐                  │
 │              │  PostgreSQL  │  Redis  │                  │
 │              └─────────────────────── ┘                  │
 │                                                          │
 │   Cockpit (admin UI · port 9090 · APT-installed)         │
 └──────────────────────────────────────────────────────────┘
          │  Tailscale VPN                │  Cloudflare Proxy
          ▼  (WireGuard · no open ports)  ▼  (HTTPS · IP hidden)
     Personal Devices                Public Internet
     (SSH · Cockpit · services)      photos.sribalaji.eu.org
                                     files.sribalaji.eu.org
```

**Two access paths — neither exposes a raw home IP:**
- **Tailscale:** Encrypted peer-to-peer VPN for admin and private access. No port forwarding required.
- **Cloudflare Proxy:** Public subdomains route through Cloudflare, hiding the home IP and providing DDoS protection and SSL.

---

## Infrastructure as Code (Ansible)

Server configuration and ongoing maintenance are managed through Ansible playbooks in a separate repo: **[iamthecloverly/ansible-homelab](https://github.com/iamthecloverly/ansible-homelab)**

The control node (MacBook) connects to the server over Tailscale SSH — no public ports opened for management.

| Playbook | What It Does |
|---|---|
| `site_hardening.yml` | UFW rules, SSH hardening, Fail2Ban setup |
| `docker_services.yml` | Container health checks, image pulls, stack restarts |
| `maintenance.yml` | System updates, Docker pruning, disk reporting |

This means the entire server config is reproducible, version-controlled, and auditable.

---

## Key Features

**Secure Remote Access**
- Tailscale VPN for all admin tasks — zero open inbound ports on the router
- SSH restricted to key-based auth; no password login
- Cockpit dashboard accessible only through the Tailscale subnet

**Custom Domain Routing**
- Cloudflare manages DNS for `sribalaji.eu.org` subdomains
- Cloudflare proxy provides HTTPS, WAF, and DDoS protection without exposing the home IP
- Optional Cloudflare Zero Trust Access for additional identity-based auth

**Encrypted Backups**
- 3-2-1 backup strategy: primary on server · secondary on external USB · off-site encrypted copy
- Client-side GPG encryption before any data leaves the server
- Immich auto-dumps its PostgreSQL database daily; Nextcloud volumes backed up alongside
- Restore process tested periodically on a VM to verify integrity

**Workflow Automation (N8N)**
- 300+ integration nodes connecting services, APIs, and custom scripts
- Example flows: nightly backup triggers, container crash alerts, Nextcloud folder watches, GitHub webhook → Telegram
- Scheduled (cron-like) and event-driven (webhooks) executions

**Telegram Bot Monitoring**
- Python bot running as a systemd service sends real-time alerts
- `/status` command returns uptime, load, and container health
- Proactive notifications: SSH logins, disk usage >90%, high CPU, container restarts
- Chat-ops style remote control (e.g., trigger a container update from Telegram)

---

## Security Layers

| Layer | Implementation |
|---|---|
| Network perimeter | UFW default-deny · only Tailscale UDP port open |
| External traffic | Cloudflare proxy · no home IP exposed |
| Admin access | Tailscale VPN · SSH key-only · Cockpit on VPN-only subnet |
| Application auth | 2FA (TOTP) on Nextcloud · key-based SSH · Fail2Ban |
| Container isolation | Non-privileged containers · DB containers on internal network only |
| Data at rest | GPG-encrypted off-site backups |
| Detection | Login alerts via Telegram · resource anomaly alerts · log review |

---

## Services at a Glance

| Service | Purpose | Port (local) |
|---|---|---|
| Immich | Photo/video management (Google Photos alternative) | 2283 |
| Nextcloud | File sync, calendar, contacts (Google Drive alternative) | 8080 |
| N8N | Workflow automation (self-hosted Zapier) | 5678 |
| Cockpit | Web-based server admin UI | 9090 |
| PostgreSQL | Shared database for Immich and Nextcloud | 5432 |
| Redis | Cache for Immich | 6379 |

---

## Maintenance Routine

- **Docker images:** Monthly pulls for minor updates; major upgrades tested against a data snapshot
- **OS patches:** `unattended-upgrades` for security fixes; full `apt upgrade` on a regular schedule
- **Backup verification:** Restore drilled in a VM every few months
- **Log review:** Monthly review of system and application logs; logrotate configured
- **Ansible drift detection:** Playbooks re-run after changes to confirm desired state

---

## Getting Started (Replication Guide)

**Requirements:**
- x86_64 machine, dual-core CPU, 8 GB RAM recommended, ample storage
- Ubuntu 24.04 LTS (headless), Docker, Docker Compose
- Free Cloudflare account (DNS + proxy), free Tailscale account
- A domain name (eu.org offers free subdomains)

**High-Level Steps:**
1. Install Ubuntu, Docker, and Cockpit (`apt install cockpit cockpit-docker`)
2. Configure UFW: default-deny, allow Tailscale UDP port (41641)
3. Install Tailscale and run `sudo tailscale up --ssh` — server is now in your Tailnet
4. Set up Cloudflare DNS with subdomain records proxied through Cloudflare
5. Write `docker-compose.yml` with PostgreSQL, Redis, Immich, Nextcloud, and N8N services
6. Store all secrets in `.env` — never hardcode in Compose files
7. Run `docker compose up -d` and complete each service's first-run wizard
8. Deploy Ansible playbooks from [ansible-homelab](https://github.com/iamthecloverly/ansible-homelab) to harden and automate the server
9. Set up Telegram bot as a systemd service for monitoring
10. Configure restic/duplicity backup jobs and schedule via cron

---

## Skills Demonstrated

- Docker and Docker Compose orchestration across multiple services
- Linux server administration (Ubuntu, systemd, UFW, APT)
- VPN and zero-trust networking (Tailscale / WireGuard)
- Infrastructure as code (Ansible playbooks and roles)
- Cloudflare DNS, proxy, and security configuration
- Automated backup strategy with encryption
- Monitoring, alerting, and incident response (Telegram, Cockpit, Fail2Ban)
- Workflow automation and API integration (N8N)
- Security hardening: defense-in-depth, least privilege, 2FA
