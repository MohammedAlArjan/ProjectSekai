# Phase 2 — Active Directory Lab Build

## Goal
Build a realistic corporate AD environment on Proxmox with intentional misconfigurations for attack practice.

## VM Specs

| VM | OS | RAM | Disk | IP |
|----|-----|-----|------|-----|
| DC01 | Windows Server 2022 | 4GB | 80GB | 10.10.10.10 |
| WS01 | Windows 10/11 | 4GB | 60GB | 10.10.10.20 |

---

## Stage 1 — Create Internal Network in Proxmox

1. Proxmox web UI → your node → **Network**
2. Click **Create** → **Linux Bridge**
3. Set:
   - Name: `vmbr1`
   - Leave **Bridge ports** empty (no physical NIC — keeps lab isolated)
   - Comment: `Lab internal network`
4. Click **Create** → **Apply Configuration**
- **Expect:** `vmbr1` appears in the network list

---

## Stage 2 — Upload Windows Server 2022 ISO

1. Proxmox web UI → **local** storage (left panel) → **ISO Images**
2. Click **Upload** → select your Windows Server 2022 ISO
3. Wait for upload to complete
- **Expect:** ISO appears in the list

---

## Stage 3 — Deploy DC01 (Domain Controller)

### Create VM
1. Click **Create VM** (top right)
2. **General:** Name: `DC01`, VM ID: `100`
3. **OS:** Select Windows Server 2022 ISO, Type: Microsoft Windows, Version: 2022
4. **System:** Leave defaults (BIOS: SeaBIOS, SCSI controller: VirtIO SCSI)
5. **Disks:** Size: `80GB`, Storage: `local-lvm`
6. **CPU:** Cores: `2`
7. **Memory:** `4096` MB
8. **Network:** Bridge: `vmbr1`, Model: VirtIO
9. Click **Finish**

### Install Windows Server 2022
1. Select DC01 → **Console** → **Start**
2. Windows Setup loads — select language, click Next
3. Select **Windows Server 2022 Standard (Desktop Experience)**
4. Accept license, select **Custom Install**
5. Select the 80GB disk → Next
6. Wait for installation (~15 minutes)
7. Set Administrator password when prompted
- **Expect:** Windows Server desktop loads

### Configure Static IP
Open PowerShell on DC01:
```powershell
# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.10.10 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.10
```

### Install AD DS and Promote to DC
```powershell
# Install AD DS role
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Promote to Domain Controller
Install-ADDSForest `
    -DomainName "sekai.local" `
    -DomainNetbiosName "SEKAI" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "YourSafeModePassword123!" -AsPlainText -Force) `
    -Force:$true
```
- **Expect:** DC01 reboots automatically, logs back in as SEKAI\Administrator

### Take Proxmox Snapshot
- Proxmox → DC01 → **Snapshots** → **Take Snapshot** → Name: `DC01-base`

---

## Stage 4 — Deploy WS01 (Workstation)

### Create VM
1. Click **Create VM**
2. **General:** Name: `WS01`, VM ID: `101`
3. **OS:** Select Windows 10/11 ISO, Type: Microsoft Windows
4. **Disks:** Size: `60GB`
5. **CPU:** Cores: `2`
6. **Memory:** `4096` MB
7. **Network:** Bridge: `vmbr1`, Model: VirtIO
8. Click **Finish**

### Install Windows 10/11
1. Start WS01 → Console
2. Install Windows normally
3. During setup — choose **Domain join instead** (skip Microsoft account)
4. Set local username: `localadmin`, password of your choice

### Configure Static IP and Join Domain
```powershell
# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.10.20 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.10

# Join domain
Add-Computer -DomainName "sekai.local" -Credential (Get-Credential) -Restart
```
- Enter `SEKAI\Administrator` credentials when prompted
- **Expect:** WS01 reboots and joins sekai.local

### Take Proxmox Snapshot
- Name: `WS01-base`

---

## Stage 5 — Create Users and Groups

Run on DC01 as Administrator:

```powershell
# Create Groups
New-ADGroup -Name "IT" -GroupScope Global -Path "CN=Users,DC=sekai,DC=local"
New-ADGroup -Name "Finance" -GroupScope Global -Path "CN=Users,DC=sekai,DC=local"
New-ADGroup -Name "Helpdesk" -GroupScope Global -Path "CN=Users,DC=sekai,DC=local"
New-ADGroup -Name "Management" -GroupScope Global -Path "CN=Users,DC=sekai,DC=local"

# Create Users
$users = @(
    @{Name="john.doe"; Group="IT"; Password="Password123!"},
    @{Name="jane.smith"; Group="Finance"; Password="Password123!"},
    @{Name="mike.jones"; Group="Helpdesk"; Password="Spring2024!"},
    @{Name="sarah.lee"; Group="Management"; Password="Winter2024!"},
    @{Name="svc_sql"; Group="IT"; Password="Password123!"},
    @{Name="svc_backup"; Group="IT"; Password="Password123!"}
)

foreach ($user in $users) {
    New-ADUser -Name $user.Name `
        -SamAccountName $user.Name `
        -UserPrincipalName "$($user.Name)@sekai.local" `
        -AccountPassword (ConvertTo-SecureString $user.Password -AsPlainText -Force) `
        -Enabled $true `
        -Path "CN=Users,DC=sekai,DC=local"
    Add-ADGroupMember -Identity $user.Group -Members $user.Name
}
```

### Make john.doe local admin on WS01
Run on WS01:
```powershell
Add-LocalGroupMember -Group "Administrators" -Member "SEKAI\john.doe"
```

---

## Stage 6 — Introduce Misconfigurations

Run on DC01 as Administrator:

### Kerberoastable Accounts (SPNs)
```powershell
Set-ADUser svc_sql -ServicePrincipalNames @{Add="MSSQLSvc/dc01.sekai.local:1433"}
Set-ADUser jane.smith -ServicePrincipalNames @{Add="HTTP/ws01.sekai.local"}
```

### ASREPRoastable (no pre-auth)
```powershell
Set-ADAccountControl mike.jones -DoesNotRequirePreAuth $true
```

### Dangerous ACLs
```powershell
# john.doe gets GenericAll over svc_sql
$user = Get-ADUser "svc_sql"
$acl = Get-Acl "AD:$($user.DistinguishedName)"
$identity = [System.Security.Principal.NTAccount]"SEKAI\john.doe"
$rights = [System.DirectoryServices.ActiveDirectoryRights]::GenericAll
$type = [System.Security.AccessControl.AccessControlType]::Allow
$rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule($identity, $rights, $type)
$acl.AddAccessRule($rule)
Set-Acl "AD:$($user.DistinguishedName)" $acl

# jane.smith gets ForceChangePassword over sarah.lee
$user = Get-ADUser "sarah.lee"
$acl = Get-Acl "AD:$($user.DistinguishedName)"
$identity = [System.Security.Principal.NTAccount]"SEKAI\jane.smith"
$rights = [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight
$type = [System.Security.AccessControl.AccessControlType]::Allow
$guid = [GUID]"00299570-246d-11d0-a768-00aa006e0529" # ForceChangePassword GUID
$rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule($identity, $rights, $type, $guid)
$acl.AddAccessRule($rule)
Set-Acl "AD:$($user.DistinguishedName)" $acl
```

### Constrained Delegation on svc_sql
```powershell
Set-ADUser svc_sql -TrustedForDelegation $false
Set-ADUser svc_sql -Add @{'msDS-AllowedToDelegateTo'=@('cifs/dc01.sekai.local')}
Set-ADAccountControl svc_sql -TrustedToAuthForDelegation $true
```

### Credentials in Share
Run on WS01:
```powershell
# Create share with hardcoded credentials script
New-Item -Path "C:\IT-Scripts" -ItemType Directory
New-SmbShare -Name "IT-Scripts" -Path "C:\IT-Scripts" -FullAccess "Everyone"
Set-Content -Path "C:\IT-Scripts\backup.ps1" -Value '# Backup script
$username = "svc_backup"
$password = "Password123!"
# Connect to backup server
net use \\dc01\backup $password /user:SEKAI\$username'
```

---

## Stage 7 — Install Sysmon on All Windows VMs

Run on DC01 and WS01:

```powershell
# Download Sysmon and config
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "C:\Sysmon.zip"
Expand-Archive "C:\Sysmon.zip" -DestinationPath "C:\Sysmon"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "C:\Sysmon\sysmonconfig.xml"

# Install
C:\Sysmon\Sysmon64.exe -accepteula -i C:\Sysmon\sysmonconfig.xml
```
- **Expect:** Sysmon service running, visible in Services

---

## Stage 8 — Final Snapshots

Take clean snapshots before any attacks:
- DC01 → Snapshots → **CLEAN**
- WS01 → Snapshots → **CLEAN**

**Phase 2 complete — AD lab is ready.**

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| WS01 can't join domain | Check DNS points to 10.10.10.10, ping dc01.sekai.local |
| AD DS install fails | Make sure static IP is set before promoting |
| Can't reach DC01 from WS01 | Both must be on vmbr1, check VM network settings |
| Sysmon install fails | Run PowerShell as Administrator |
