# Phase 2 — Active Directory Lab Build

## Goal
Build a realistic corporate AD environment on Proxmox with intentional misconfigurations.

---

## VM Specs

| VM | OS | RAM | Disk | IP |
|----|-----|-----|------|-----|
| DC01 | Windows Server 2022 | 4GB | 80GB | 10.10.10.10 |
| WS01 | Windows 10/11 | 4GB | 60GB | 10.10.10.20 |

---

## Step 1 — Create Internal Network in Proxmox
- [ ] Create a new Linux Bridge in Proxmox (e.g. `vmbr1`) — isolated lab network
- [ ] Assign subnet: `10.10.10.0/24`
- [ ] Do NOT bridge to physical NIC (keep lab isolated)

## Step 2 — Deploy DC01 (Domain Controller)
- [ ] Create VM in Proxmox, attach Windows Server 2022 ISO
- [ ] Install Windows Server 2022 (Desktop Experience)
- [ ] Set static IP: `10.10.10.10`
- [ ] Install AD DS role
- [ ] Promote to Domain Controller
- [ ] Domain: `sekai.local`
- [ ] Set DNS to `10.10.10.10` (itself)

## Step 3 — Deploy WS01 (Workstation)
- [ ] Create VM, attach Windows 10/11 ISO
- [ ] Install Windows
- [ ] Set static IP: `10.10.10.20`
- [ ] Set DNS to `10.10.10.10`
- [ ] Join `sekai.local` domain

## Step 4 — Create Users and Groups
Create the following structure:

**Groups:**
- IT
- Finance
- Helpdesk
- Management

**Users:**
| Username | Group | Notes |
|----------|-------|-------|
| john.doe | IT | Local admin on WS01 |
| jane.smith | Finance | Kerberoastable |
| mike.jones | Helpdesk | ASREPRoastable |
| sarah.lee | Management | Weak password |
| svc_sql | IT | Has SPN, Kerberoastable |
| svc_backup | IT | Has SPN |
| Administrator | Domain Admins | Strong password |

## Step 5 — Introduce Misconfigurations

### Weak Passwords
- [ ] Set `sarah.lee` password to `Winter2024!`
- [ ] Set `mike.jones` password to `Spring2024!`

### Kerberoastable Accounts
- [ ] Set SPN on `svc_sql`: `MSSQLSvc/dc01.sekai.local:1433`
- [ ] Set SPN on `jane.smith`: `HTTP/ws01.sekai.local`

### ASREPRoastable
- [ ] Disable pre-auth on `mike.jones`

### Dangerous ACLs
- [ ] Give `john.doe` GenericAll over `svc_sql`
- [ ] Give `jane.smith` ForceChangePassword over `sarah.lee`
- [ ] Give `svc_backup` WriteDACL on Domain object

### Delegation
- [ ] Set constrained delegation on `svc_sql` pointing to `cifs/dc01.sekai.local`

### Credentials in Shares
- [ ] Create a share on WS01 with a script containing hardcoded credentials

## Step 6 — Install Sysmon on All Windows VMs
```powershell
# Download Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
Expand-Archive Sysmon.zip

# Download SwiftOnSecurity config
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "sysmonconfig.xml"

# Install
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```
- [ ] Install Sysmon on DC01
- [ ] Install Sysmon on WS01

---

## Notes
- Take Proxmox snapshots after each major step — easy to revert after attacks
- Keep a clean snapshot named `CLEAN` before any attacks
- Windows Server 2022 evaluation is free for 180 days
