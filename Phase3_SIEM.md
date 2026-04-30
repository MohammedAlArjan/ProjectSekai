# Phase 3 — SIEM Setup (Wazuh)

## Goal
Deploy Wazuh SIEM and collect logs from all lab VMs for attack detection.

---

## VM Specs
| VM | OS | RAM | Disk | IP |
|----|-----|-----|------|-----|
| Wazuh | Ubuntu 22.04 | 8GB | 100GB | 10.10.10.30 |

---

## Step 1 — Deploy Wazuh VM
- [ ] Create VM in Proxmox with Ubuntu 22.04 ISO
- [ ] Set static IP: `10.10.10.30`
- [ ] Update system: `apt update && apt upgrade -y`

## Step 2 — Install Wazuh (All-in-One)
```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```
- [ ] Installation completes successfully
- [ ] Note the admin password printed at the end
- [ ] Access Wazuh dashboard: `https://10.10.10.30`

## Step 3 — Install Wazuh Agent on DC01
```powershell
# Run on DC01
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi" -OutFile "wazuh-agent.msi"
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="10.10.10.30" WAZUH_AGENT_NAME="DC01"
NET START WazuhSvc
```
- [ ] Agent installed on DC01
- [ ] DC01 appears in Wazuh dashboard as active

## Step 4 — Install Wazuh Agent on WS01
```powershell
# Run on WS01
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi" -OutFile "wazuh-agent.msi"
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="10.10.10.30" WAZUH_AGENT_NAME="WS01"
NET START WazuhSvc
```
- [ ] Agent installed on WS01
- [ ] WS01 appears in Wazuh dashboard as active

## Step 5 — Configure Windows Event Log Collection
Add to `ossec.conf` on each Windows agent:
```xml
<localfile>
  <location>Security</location>
  <log_format>eventchannel</log_format>
</localfile>
<localfile>
  <location>System</location>
  <log_format>eventchannel</log_format>
</localfile>
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```
- [ ] Security logs flowing into Wazuh
- [ ] Sysmon logs flowing into Wazuh

## Step 6 — Verify Detection
- [ ] Run a test: failed login attempt on DC01
- [ ] Verify alert appears in Wazuh dashboard
- [ ] Phase 3 complete — take Proxmox snapshot

---

## Key Windows Event IDs to Watch
| Event ID | Description |
|----------|-------------|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4648 | Logon with explicit credentials |
| 4662 | Object access (DCSync) |
| 4768 | Kerberos TGT request |
| 4769 | Kerberos service ticket request (Kerberoast) |
| 4771 | Kerberos pre-auth failed (ASREPRoast) |
| 4776 | NTLM authentication |
| 7045 | New service installed |
