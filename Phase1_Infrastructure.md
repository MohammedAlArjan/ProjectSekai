# Phase 1 — Infrastructure Setup

## Goal
Get Proxmox running on PC A and accessible remotely from PC B via Tailscale.

## Storage Plan
- **Install target:** 1TB M.2 (Proxmox OS + all VMs)
- **250GB SSD:** spare, add to Proxmox later if needed
- **2TB HDD:** disconnect before installing — contains important files, do not touch

---

## Stage 1 — Prepare USB (on PC B)

- [ ] Download Proxmox VE ISO: `https://www.proxmox.com/en/downloads`
- [ ] Download Rufus: `https://rufus.ie`
- [ ] Plug in USB drive (8GB minimum)
- [ ] Open Rufus → select USB → select Proxmox ISO → click Start
- **Expect:** Rufus writes ISO in 2-5 minutes

---

## Stage 2 — Install Proxmox on PC A

- [ ] **Disconnect the 2TB HDD before proceeding**
- [ ] Plug USB into PC A and boot
- [ ] Enter BIOS (F2/F12/Del) → set USB as first boot device, disable Secure Boot
- [ ] Proxmox installer loads → select **Install Proxmox VE**
- [ ] Accept license
- [ ] Select **1TB M.2** as target disk
- [ ] Set country, timezone, keyboard
- [ ] Set root password (save this somewhere safe)
- [ ] Set email (can be fake)
- [ ] **Network config:**
  - Hostname: `proxmox.sekai.local`
  - IP: static IP on your local network (e.g. `192.168.1.100`)
  - Gateway: your router IP (e.g. `192.168.1.1`)
  - DNS: `8.8.8.8`
- [ ] Click Install — takes 5-10 minutes
- [ ] Remove USB when prompted, PC A reboots
- **Expect:** Black screen showing Proxmox is running and the web UI address

---

## Stage 3 — First Access to Proxmox

- [ ] From PC B browser: `https://192.168.1.100:8006`
- [ ] Accept certificate warning → click Advanced → Proceed
- [ ] Login: `root` / (password you set)
- **Expect:** Proxmox dashboard, no VMs yet, subscription warning (ignore it)

---

## Stage 4 — Update Proxmox

In Proxmox web UI → click node (left panel) → Shell:

```bash
# Disable enterprise repo (requires paid subscription)
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Add free community repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" >> /etc/apt/sources.list

# Update
apt update && apt full-upgrade -y
```

- [ ] Updates complete with no errors
- [ ] Reboot Proxmox after update

---

## Stage 5 — Install Tailscale on PC A

In Proxmox shell:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

- [ ] A URL appears in terminal — open it on PC B browser
- [ ] Authenticate with your Tailscale account
- [ ] PC A gets a Tailscale IP (100.x.x.x)
- **Expect:** PC A appears green in Tailscale dashboard

---

## Stage 6 — Connect PC B via Tailscale

- [ ] Download Tailscale on PC B: `https://tailscale.com/download`
- [ ] Login with same Tailscale account as PC A
- **Expect:** Both PC A and PC B appear green in Tailscale dashboard

---

## Stage 7 — Verify Remote Access via Tailscale

- [ ] From PC B browser: `https://<PC-A-Tailscale-IP>:8006`
- [ ] Login as root successfully
- **Expect:** Full Proxmox dashboard over Tailscale — Phase 1 complete

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Can't boot from USB | Check BIOS boot order, disable Secure Boot |
| Wrong disk selected | Installer shows disk sizes — pick the 1TB one |
| Can't reach web UI | Ping `192.168.1.100` from PC B, check PC A IP |
| Tailscale URL expired | Run `tailscale up` again in Proxmox shell |
| Certificate warning in browser | Normal — click Advanced → Proceed |

---

## Notes
- Proxmox login: `root` / password set during install
- Keep PC A's local IP static — set a reservation in your router by MAC address
- Tailscale free tier supports up to 100 devices
