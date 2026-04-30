# Project Sekai — Home Cybersecurity Lab

## Overview
A full home lab simulating a corporate Active Directory environment, attacked and monitored end-to-end.
Built for portfolio and LinkedIn/GitHub showcase.

## Infrastructure
| Machine | Role | Specs |
|---------|------|-------|
| PC A (Lab Server) | Proxmox hypervisor hosting all VMs | i5-9600K, 32GB DDR4, 3TB, RTX 2060 |
| PC B (Attack Machine) | Kali Linux daily driver | Ryzen 9800X3D, 32GB DDR5, RTX 9070 XT |

**Network:** PC A and PC B connected via Tailscale VPN across separate networks.

## Domain
`sekai.local`

---

## Project Phases

### Phase 1 — Infrastructure Setup
- [ ] Wipe PC A and install Proxmox
- [ ] Configure Proxmox networking
- [ ] Install Tailscale on PC A
- [ ] Connect PC A and PC B via Tailscale
- [ ] Verify remote access to Proxmox web UI from PC B

### Phase 2 — Active Directory Lab Build
- [ ] Deploy Windows Server 2022 VM (DC01.sekai.local)
- [ ] Deploy Windows 10/11 VM (WS01 — workstation)
- [ ] Deploy Kali Linux VM (optional internal attacker)
- [ ] Configure AD domain: sekai.local
- [ ] Create users, groups, OUs
- [ ] Introduce intentional misconfigurations (ACL abuse, delegation, weak passwords, Kerberoastable accounts)

### Phase 3 — SIEM Setup (Wazuh)
- [ ] Deploy Wazuh VM on Proxmox
- [ ] Install Wazuh agents on DC01 and WS01
- [ ] Configure log collection (Windows Event Logs, Sysmon)
- [ ] Deploy Sysmon with SwiftOnSecurity config on all Windows VMs
- [ ] Verify logs are flowing into Wazuh dashboard

### Phase 4 — Attack & Detect
- [ ] Perform attacks from PC B (Kali) against sekai.local
- [ ] Document each attack with screenshots and commands
- [ ] Verify detections in Wazuh for each attack
- [ ] Write detection rules for missed attacks
- [ ] Document findings as a pentest report

### Phase 5 — Portfolio Packaging
- [ ] Write GitHub README with architecture diagram
- [ ] Write attack walkthrough writeups
- [ ] Create LinkedIn post summarizing the project
- [ ] Record optional demo video

---

## VM Resource Allocation (PC A — 32GB RAM, 3TB)

| VM | OS | RAM | Disk |
|----|-----|-----|------|
| DC01 | Windows Server 2022 | 4GB | 80GB |
| WS01 | Windows 10/11 | 4GB | 60GB |
| Wazuh | Ubuntu 22.04 | 8GB | 100GB |
| Kali (optional) | Kali Linux | 4GB | 60GB |
| **Total** | | **20GB** | **300GB** |

12GB RAM headroom for Proxmox and future VMs.

---

## Potential Scale (Future Phases)

### More Attack Surface
| VM | OS | RAM | Disk | Purpose |
|----|-----|-----|------|---------|
| FS01 | Windows Server 2022 | 4GB | 80GB | File server with sensitive shares |
| WS02 | Windows 10/11 | 4GB | 60GB | Second workstation, lateral movement |
| MSSQL01 | Windows Server 2022 | 4GB | 80GB | SQL Server, MSSQL attack practice |
| ADCS01 | Windows Server 2022 | 4GB | 80GB | Certificate Authority, ESC1-ESC8 attacks |
| Linux01 | Ubuntu 22.04 | 2GB | 40GB | Mixed environment, Linux privesc |

### More Defense
| VM | OS | RAM | Disk | Purpose |
|----|-----|-----|------|---------|
| Velociraptor | Ubuntu 22.04 | 4GB | 100GB | Advanced DFIR alongside Wazuh |
| Honeypot | Any | 2GB | 40GB | Catch attackers with fake credentials |

### Additional Attack Scenarios (on scale)
- ADCS abuse (ESC1-ESC8) via Certipy
- MSSQL attacks (xp_cmdshell, linked servers, NTLM capture)
- Mixed Windows/Linux lateral movement
- Honeypot evasion and detection

---

## Intentional Misconfigurations to Introduce (Phase 2)
- Weak passwords following SeasonYear! format
- Kerberoastable service accounts (SPNs on user accounts)
- ASREPRoastable accounts (no pre-auth)
- Dangerous ACLs (GenericAll, ForceChangePassword, WriteDACL)
- Unconstrained/constrained delegation
- SMB signing disabled on workstation
- Credentials in shares/scripts

---

## Attack Scenarios to Cover (Phase 4)
- Password spraying
- ASREPRoasting
- Kerberoasting
- BloodHound enumeration
- ACL abuse (ForceChangePassword, GenericAll)
- Constrained delegation abuse (S4U2self + S4U2proxy)
- DCSync
- Pass-the-Hash
- NTLM relay (if SMB signing disabled)
- Credential hunting (shares, scripts, registry)

---

## Tools Used
**Attack:** Nmap, Netexec, BloodHound, Impacket suite, Certipy, bloodyAD, Rubeus, Mimikatz, Hashcat
**Defense:** Wazuh SIEM, Sysmon, Windows Event Logs
