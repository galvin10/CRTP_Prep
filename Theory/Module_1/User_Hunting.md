# Domain Enumeration — User Hunting & Session Detection

**Date:** 25 July 2026

---

## Task 1: Find Machines Where Current User Has Local Admin Access

```powershell
Find-LocalAdminAccess -Verbose
```

Queries DC for all computers and checks local admin access on each. Uses multi-threaded `Invoke-CheckLocalAdminAccess`.

**Alternative methods (if RPC/SMB ports blocked):**
```powershell
Find-WMILocalAdminAccess.ps1
Find-PSRemotingLocalAdminAccess.ps1
```

---

## Task 2: Find Computers Where Domain Admin Has Sessions

```powershell
Find-DomainUserLocation -Verbose
```

Default: searches for Domain Admins sessions. Queries DC for group members, gets computers list, and checks sessions/logged-on users.

**Filter by specific group:**
```powershell
Find-DomainUserLocation -UserGroupIdentity "RDPUsers"
```

> **Note:** Server 2019+ requires admin privileges to list sessions.

---

## Task 3: Find Domain Admin Sessions Where Current User Has Admin Access

```powershell
Find-DomainUserLocation -CheckAccess
```

Combines session hunting with admin access check using `Test-AdminAccess`.

---

## Task 4: Stealthy Session Hunting (File Servers Only)

```powershell
Find-DomainUserLocation -Stealth
```

Targets file servers and distributed file servers only (fewer machines to check).

---

## Task 5: Remote Registry-based Session Hunting (No Admin Required)

```powershell
Invoke-SessionHunter -FailSafe
```

Uses Remote Registry to query HKEY_USERS hive. Does not require admin access on remote machines.

**OPSEC-friendly (specify targets):**
```powershell
Invoke-SessionHunter -NoPortScan -Targets C:\AD\Tools\servers.txt
```

Avoids port scanning and connecting to all machines. Only checks specified servers.

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [Invoke-SessionHunter](https://github.com/Leo4j/Invoke-SessionHunter)

---

*Next: Learning Objective 5 — Kerberoasting & Credential Extraction*
