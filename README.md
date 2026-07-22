# 🛡️ BlueWatch SOC

**A fully containerized, open-source Security Operations Center (SOC) lab** built to detect, correlate, and respond to real-world attack scenarios in real time — integrating SIEM, XDR, and NIDS capabilities on a single platform.

![Wazuh](https://img.shields.io/badge/SIEM%2FXDR-Wazuh-1abc9c)
![Suricata](https://img.shields.io/badge/NIDS-Suricata-orange)
![Docker](https://img.shields.io/badge/Deployed%20with-Docker-2496ED)
![Kali](https://img.shields.io/badge/Agent%2FAttacker-Kali%20Linux-557C94)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

---

## 📖 Overview

BlueWatch SOC is a home-lab Security Operations Center built from scratch to practice real blue-team workflows: deploying a SIEM, tuning detection rules, monitoring endpoints, and mapping alerts to the MITRE ATT&CK framework. The lab simulates attacker behavior on a monitored endpoint and verifies that the detection pipeline (agent → manager → dashboard) captures, enriches, and displays the resulting alerts correctly.

## 🧱 Architecture

```
                     ┌─────────────────────────────┐
                     │        Windows Host          │
                     │  (Docker Desktop + VMware)    │
                     │                              │
                     │  ┌────────────────────────┐  │
                     │  │   Wazuh Manager (SIEM)  │  │
                     │  │   + Wazuh Indexer       │  │
                     │  │   + Wazuh Dashboard     │  │
                     │  │        (Docker)         │  │
                     │  └───────────▲────────────┘  │
                     │              │ 1514/1515      │
                     │              │ (encrypted)     │
                     └──────────────┼────────────────┘
                                    │
                     ┌──────────────▼────────────────┐
                     │      Kali Linux (VMware VM)     │
                     │   - Wazuh Agent                 │
                     │   - Suricata (NIDS)              │
                     │   - Auditd                        │
                     │   - Acts as monitored endpoint    │
                     │     AND attack source             │
                     └────────────────────────────────┘
```

## 🛠️ Tech Stack

| Component | Role |
|---|---|
| **Wazuh** | SIEM / XDR — log collection, correlation, dashboards, active response |
| **Suricata** | Network Intrusion Detection (NIDS) — deep packet inspection |
| **Auditd** | Linux syscall/command auditing fed into Wazuh |
| **Docker / Docker Desktop** | Hosts the Wazuh manager, indexer, and dashboard stack |
| **Kali Linux (VMware)** | Monitored endpoint (Wazuh agent) and attack-simulation source |
| **MITRE ATT&CK** | Framework used to map and classify detected techniques |

## 🔐 Key Capabilities Implemented

- **Automated Active Response** — auto-blocks source IPs performing SSH brute-force attempts.
- **File Integrity Monitoring (FIM)** — real-time monitoring of critical directories (`/etc`, `/bin`) for unauthorized changes, additions, or persistence attempts.
- **Network Intrusion Detection** — Suricata rules detect suspicious traffic patterns (ICMP floods, port scans, known-bad user-agents) and forward alerts into Wazuh.
- **Malicious Command / Recon Detection** — Auditd rules flag suspicious command execution (e.g. reconnaissance tools), mapped to MITRE ATT&CK tactics such as *Execution* and *Persistence*.
- **Vulnerability Management** — Wazuh's Syscollector module inventories installed packages and cross-references known CVEs automatically.
- **MITRE ATT&CK Mapping** — every relevant alert is tagged with its corresponding tactic/technique ID for faster triage.

## 🖼️ Screenshots

### 1. Overview Dashboard — Agent Status & Alert Severity
Live agent connectivity status and a 24-hour breakdown of alerts by severity, alongside quick links to Endpoint Security and Threat Intelligence modules.

![Wazuh Overview Dashboard](docs/screenshots/01-wazuh-overview-dashboard.jpeg)

### 2. Attack Simulation on Monitored Endpoint
Commands run on the Kali agent to generate telemetry for the detection pipeline: config edits, service restarts, a monitored file write, and outbound traffic with a flagged user-agent string.

![Attack Simulation Terminal](docs/screenshots/02-kali-attack-simulation-terminal.jpeg)

### 3. MITRE ATT&CK — Correlated Events
Alerts automatically enriched with MITRE tactic and technique IDs (e.g. `T1548.003`, `T1078`, `T1565.001`), enabling tactic-level triage instead of raw rule IDs.

![MITRE ATT&CK Events](docs/screenshots/03-mitre-attck-events.jpeg)

### 4. Threat Hunting Dashboard
Aggregated view of alert-level evolution over time and the top MITRE ATT&CK techniques observed across the environment.

![Threat Hunting Dashboard](docs/screenshots/04-threat-hunting-dashboard.jpeg)

### 5. Threat Hunting — Raw Event Table
Full event list showing rule descriptions triggered during the exercise — FIM checksum changes, Auditd command executions, Suricata network alerts, and PAM session/sudo events.

![Threat Hunting Events](docs/screenshots/05-threat-hunting-events.jpeg)

### 6. Discover — FIM & Auditd Log Detail
Deep-dive into a single FIM alert showing full `syscheck` fields (before/after hash, permissions, ownership) next to a raw Auditd `SYSCALL` record.

![Discover FIM & Audit Logs](docs/screenshots/06-discover-fim-audit-logs.jpeg)

## 🚀 Getting Started

Full step-by-step setup (Docker install, Wazuh deployment, agent enrollment, Suricata/Auditd config) lives in:

➡️ **[docs/installation.md](docs/installation.md)**

## 💡 Challenges & Lessons Learned

The biggest hurdle was networking between the Dockerized manager (on the Windows host) and the virtualized agent (Kali on VMware). Getting reliable, encrypted agent↔manager communication on ports **1514/1515** required working through:
- NAT vs. Bridged adapter mode on the VM
- Windows Firewall rules blocking inbound Docker traffic
- Correct manager IP registration during agent enrollment

## 🎯 Skills Demonstrated

`SIEM Deployment & Configuration` · `SOC Monitoring & Log Analysis` · `Threat Detection & Alert Correlation` · `IDS/IPS (Suricata)` · `Docker Containers & Networking` · `Endpoint Security Monitoring` · `Linux Security & Hardening` · `Incident Response Fundamentals`

## 🤝 Contact

Interested in SOC projects, blue-team learning paths, or Wazuh setups? Feel free to open an issue or reach out — always happy to collaborate and learn together.

---
*This is a personal home-lab project built for learning purposes on isolated, self-owned virtual machines. No production systems or third-party assets were targeted.*
