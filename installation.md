# ⚙️ Installation Guide — BlueWatch SOC

This guide walks through standing up the lab exactly as pictured in the screenshots:
a **Dockerized Wazuh manager stack on Windows**, and a **Kali Linux VM (VMware)** running the Wazuh agent, Suricata, and Auditd.

> Tested with: Docker Desktop (Windows), Wazuh 4.x, VMware Workstation, Kali Linux 2025.1a.

---

## 1. Prerequisites

- Windows 10/11 with **Docker Desktop** installed and running (WSL2 backend recommended)
- **VMware Workstation** (or Player) with a Kali Linux VM
- At least 4 GB RAM / 2 vCPUs free for the Wazuh stack
- Network mode on the Kali VM set to **Bridged** (simplest option) or NAT with port forwarding configured for 1514/1515

---

## 2. Deploy the Wazuh Manager Stack (Docker — on Windows host)

Clone the official Wazuh Docker repository, which ships pre-built compose files for the manager, indexer, and dashboard:

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.9.2
cd wazuh-docker/single-node
```

Generate the required SSL certificates (used for indexer ↔ dashboard ↔ manager TLS):

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

Start the stack:

```bash
docker compose up -d
```

Verify all three containers are healthy:

```bash
docker ps
```

You should see `wazuh.manager`, `wazuh.indexer`, and `wazuh.dashboard` running. The dashboard is now reachable at:

```
https://localhost:443
```

Default credentials (change immediately after first login):

```
Username: admin
Password: SecretPassword
```

> ⚠️ Always rotate the default password and indexer/dashboard certs before exposing this beyond an isolated lab network.

---

## 3. Install the Wazuh Agent on Kali Linux

On the **Kali VM**, add the Wazuh repository and install the agent:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list

sudo apt update
sudo apt install wazuh-agent -y
```

During install (or afterward via `/var/ossec/etc/ossec.conf`), point the agent at your manager's IP:

```bash
sudo sed -i "s/MANAGER_IP/<your-docker-host-ip>/" /var/ossec/etc/ossec.conf
```

Enable and start the agent:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Confirm connectivity from the dashboard: **Agents Summary → Active (1)** should appear, as shown in the Overview screenshot.

---

## 4. Configure File Integrity Monitoring (FIM)

Edit the agent config to watch sensitive directories:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add/confirm a `syscheck` block like:

```xml
<syscheck>
  <directories realtime="yes" report_changes="yes" check_all="yes">/etc,/bin,/sbin,/usr/bin</directories>
  <frequency>43200</frequency>
</syscheck>
```

Restart the agent to apply:

```bash
sudo systemctl restart wazuh-agent
```

Any create/modify/delete under those paths (e.g. a new file appearing in `/etc`) now generates a `syscheck` alert, visible under **Endpoint Security → File Integrity Monitoring** and in **Discover** (see screenshot 6).

---

## 5. Install & Integrate Suricata (NIDS)

On the Kali VM:

```bash
sudo apt install suricata -y
sudo suricata-update
sudo systemctl enable suricata
sudo systemctl start suricata
```

Point Wazuh at Suricata's EVE JSON log by adding a `localfile` block in `ossec.conf`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Restart the agent:

```bash
sudo systemctl restart wazuh-agent
```

Suricata alerts (e.g. flagged user-agents, port scans, ICMP floods) now flow into Wazuh and appear in **Threat Hunting** and **Discover** with the `data.alert.*` fields populated.

---

## 6. Enable Auditd for Command-Level Visibility

```bash
sudo apt install auditd audispd-plugins -y
sudo systemctl enable auditd
sudo systemctl start auditd
```

Add audit rules to track execution of common recon/attacker tools:

```bash
sudo auditctl -w /usr/bin/nc.traditional -p x -k recon_tools
sudo auditctl -w /usr/bin/curl -p x -k recon_tools
```

Persist rules across reboots in `/etc/audit/rules.d/audit.rules`, then make sure Wazuh is reading the audit log:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

These executions surface in Wazuh as `Audit: Command: ...` alerts (rule ID `80792`), correlated with MITRE ATT&CK techniques where applicable.

---

## 7. Enable Active Response (SSH Brute-Force Auto-Block)

On the **manager** side (inside the `wazuh.manager` container or its mounted config), enable the built-in `firewall-drop` active response tied to the SSHD brute-force rule group:

```xml
<active-response>
  <disabled>no</disabled>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763,5760</rules_id>
  <timeout>600</timeout>
</active-response>
```

Restart the manager container to apply:

```bash
docker restart single-node-wazuh.manager-1
```

Repeated failed SSH logins from the same source now trigger an automatic firewall block for the configured timeout window.

---

## 8. Verify the Pipeline End-to-End

From the Kali VM, generate some test telemetry (safe, self-contained — targets your own lab only):

```bash
sudo touch /etc/secret.txt          # triggers FIM alert
nc -z localhost 22                  # triggers Auditd command alert
curl -A "<test-user-agent-string>" http://<your-test-endpoint>/   # triggers Suricata alert if a matching rule exists
```

Then check:
- **Overview** → alert counts should tick up
- **Threat Hunting → Events** → new rows for `syscheck`, Auditd, and Suricata rule IDs
- **MITRE ATT&CK → Events** → any rules with `rule.mitre.id` populated appear here automatically

---

## 9. Useful Troubleshooting Commands

```bash
# Check agent connection status from the manager container
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/agent_control -l

# Tail manager logs
docker logs -f single-node-wazuh.manager-1

# Tail agent logs on Kali
sudo tail -f /var/ossec/logs/ossec.log

# Confirm ports 1514 (agent events) and 1515 (enrollment) are reachable from the VM
nc -zv <manager-ip> 1514
nc -zv <manager-ip> 1515
```

Common fixes:
- **Agent stuck "Disconnected"** → check VM network mode (Bridged is simplest), Windows Firewall rules for Docker, and that the manager IP in `ossec.conf` is correct.
- **No FIM alerts** → confirm `realtime="yes"` and that the agent was restarted after editing `ossec.conf`.
- **No Suricata alerts in Wazuh** → verify the `eve.json` path and that `suricata-update` pulled a rule set.

---

## 📎 Related

- Wazuh official docs: https://documentation.wazuh.com/
- Wazuh Docker deployment: https://github.com/wazuh/wazuh-docker
- Suricata docs: https://docs.suricata.io/
- MITRE ATT&CK: https://attack.mitre.org/
