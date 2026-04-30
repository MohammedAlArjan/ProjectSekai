# Project Sekai — Network Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PC B (Attack Machine)                        │
│                    Ryzen 9800X3D | 32GB DDR5                        │
│                         Kali Linux                                   │
│                    Tailscale IP: 100.x.x.2                          │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                    [ Tailscale VPN ]
                    (across networks)
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│                         PC A (Lab Server)                            │
│                    i5-9600K | 32GB DDR4 | 3TB                       │
│                         Proxmox VE                                   │
│                    Tailscale IP: 100.x.x.1                          │
│                    Web UI: https://100.x.x.1:8006                   │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │              Internal Lab Network (vmbr1)                     │  │
│   │                    10.10.10.0/24                              │  │
│   │                                                               │  │
│   │   ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐  │  │
│   │   │    DC01      │   │    WS01     │   │     Wazuh       │  │  │
│   │   │ Windows     │   │ Windows     │   │  Ubuntu 22.04   │  │  │
│   │   │ Server 2022 │   │  10/11      │   │   SIEM/IDS      │  │  │
│   │   │             │   │             │   │                 │  │  │
│   │   │ 10.10.10.10 │   │ 10.10.10.20 │   │  10.10.10.30   │  │  │
│   │   │             │   │             │   │                 │  │  │
│   │   │ - AD DS     │   │ - Domain    │   │ - Wazuh Manager │  │  │
│   │   │ - DNS       │   │   joined    │   │ - Dashboard     │  │  │
│   │   │ - Kerberos  │   │ - Sysmon    │   │ - Alerts        │  │  │
│   │   │ - Sysmon    │   │ - Wazuh     │   │                 │  │  │
│   │   │ - Wazuh     │   │   Agent     │   │                 │  │  │
│   │   │   Agent     │   │             │   │                 │  │  │
│   │   └──────┬──────┘   └──────┬──────┘   └────────┬────────┘  │  │
│   │          │                 │                    │            │  │
│   │          └─────────────────┴────────────────────┘            │  │
│   │                     10.10.10.0/24                             │  │
│   └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

Attack Flow:
PC B (Kali) ──[Tailscale]──► Proxmox ──[vmbr1]──► DC01 / WS01

Log Flow:
DC01 ──► Wazuh Agent ──► Wazuh Manager (10.10.10.30)
WS01 ──► Wazuh Agent ──► Wazuh Manager (10.10.10.30)

Domain: sekai.local
```
