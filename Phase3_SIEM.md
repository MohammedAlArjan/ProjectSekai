# Phase 3 — SIEM Setup (Wazuh)

## Goal
Deploy Wazuh SIEM and collect logs from all lab VMs for attack detection.

---

## VM Specs
| VM | OS | RAM | Disk | IP |
|----|-----|-----|------|-----|
| Wazuh | Ubuntu 22.04 LTS Server | 8GB | 100GB | 10.10.10.30 |

---

## Stage 1 — Upload Ubuntu 22.04 ISO

1. Download Ubuntu Server 22.04 LTS ISO on PC B: `https://ubuntu.com/download/server`
2. Proxmox web UI → **local** storage → **ISO Images** → **Upload**
3. Select the Ubuntu ISO → wait for upload
- **Expect:** ISO appears in the list

---

## Stage 2 — Create Wazuh VM in Proxmox

1. Click **Create VM**
2. **General:**
   - Name: `Wazuh`
   - VM ID: `102`
3. **OS:**
   - ISO: select Ubuntu 22.04 ISO
   - Type: **Linux**
   - Version: **6.x - 2.6 Kernel**
4. **System:** leave defaults
5. **Disks:**
   - Bus: **SATA**
   - Size: **100GB**
   - Storage: **local-lvm**
6. **CPU:**
   - Sockets: 1
   - Cores: **2**
   - Type: **host**
7. **Memory:** `8192` MB
8. **Network:**
   - Bridge: **vmbr1**
   - Model: **Intel E1000**
9. Click **Finish**

---

## Stage 3 — Install Ubuntu Server

1. Start Wazuh VM → Console
2. Select **Try or Install Ubuntu Server**
3. Select language: English
4. Keyboard layout: your preference
5. Installation type: **Ubuntu Server** (not minimized)
6. Network — skip for now (configure after)
7. No proxy
8. Keep default mirror
9. Storage: **Use entire disk** → select the 100GB disk
10. Profile setup:
    - Your name: `wazuh`
    - Server name: `wazuh`
    - Username: `wazuh`
    - Password: something you'll remember (e.g. `Admin123!`)
11. **Skip** Ubuntu Pro
12. Check **Install OpenSSH server** — useful for remote access
13. No snaps needed → Done
14. Wait for installation (~10 minutes) → Reboot
15. Remove ISO after reboot: Proxmox → Wazuh → Hardware → CD/DVD Drive → **Do not use any media**
- **Expect:** Ubuntu login prompt

---

## Stage 4 — Configure Static IP

Login as `wazuh` then run:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace contents with:

```yaml
network:
  ethernets:
    ens18:
      addresses:
        - 10.10.10.30/24
      nameservers:
        addresses:
          - 10.10.10.10
      routes:
        - to: default
          via: 10.10.10.1
  version: 2
```

Apply:

```bash
sudo netplan apply
```

Verify:

```bash
ip a
ping 10.10.10.10
```

- **Expect:** IP set to 10.10.10.30, ping to DC01 succeeds

---

## Stage 5 — Give Wazuh Internet Access (Temporary)

Wazuh needs internet to download its installer.

1. Proxmox → Wazuh → Hardware → **Add** → **Network Device**
2. Bridge: `vmbr0`, Model: Intel E1000 → **Add**
3. Inside Wazuh VM:

```bash
sudo dhclient ens19
```

Test internet:

```bash
ping 8.8.8.8
```

---

## Stage 6 — Install Wazuh (All-in-One)

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

- Installation takes 10-20 minutes
- **At the end it prints the admin username and password — save these**
- **Expect:** Installation completes with no errors

Remove internet access when done:
1. Proxmox → Wazuh → Hardware → select `vmbr0` network device → **Remove**

---

## Stage 7 — Access Wazuh Dashboard

From PC B browser:

```
https://10.10.10.30
```

- Login with the credentials printed during installation
- **Expect:** Wazuh dashboard loads

---

## Stage 8 — Install Wazuh Agent on DC01

Run on DC01 (PowerShell as Administrator):

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi" -OutFile "wazuh-agent.msi"
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="10.10.10.30" WAZUH_AGENT_NAME="DC01"
NET START WazuhSvc
```

- [ ] DC01 appears in Wazuh dashboard as active

---

## Stage 9 — Install Wazuh Agent on WS01

Run on WS01 (PowerShell as Administrator):

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi" -OutFile "wazuh-agent.msi"
msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="10.10.10.30" WAZUH_AGENT_NAME="WS01"
NET START WazuhSvc
```

- [ ] WS01 appears in Wazuh dashboard as active

---

## Stage 10 — Configure Windows Event Log Collection

On DC01 and WS01, add to `C:\Program Files (x86)\ossec-agent\ossec.conf` inside the `<ossec_config>` block:

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

Restart agent after editing:

```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

---

## Stage 11 — Verify Detection

1. On WS01 — attempt a failed login: lock screen → wrong password 3 times
2. Go to Wazuh dashboard → **Security Events**
3. Look for Event ID 4625 alert
- **Expect:** Alert appears within 30 seconds

---

## Stage 12 — Final Snapshot

- Proxmox → Wazuh → Snapshots → **Take Snapshot** → Name: `Wazuh-base`
- **Phase 3 complete**

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Can't reach Wazuh dashboard | Check IP is 10.10.10.30, check vmbr1 |
| Agent not showing in dashboard | Check WAZUH_MANAGER IP, restart WazuhSvc |
| No logs in dashboard | Check ossec.conf edits, restart agent |
| Netplan interface name wrong | Run `ip a` to find correct interface name |

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
