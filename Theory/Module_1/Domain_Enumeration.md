# Domain Enumeration

**Date:** 19 July 2026

> Comprehensive notes on enumerating Active Directory objects, users, computers, groups, and shares. Includes both theory and hands-on lab commands using PowerView, AD Module, and BloodHound.

---

## 1. Domain Enumeration — Theory

### Tools & Detection

**Detection Landscape**
- Microsoft Defender for Endpoint (MDE) detects most enumeration tools
- MDE actively monitors LDAP queries sent during enumeration
- BloodHound queries can trigger MDE detection
- Tools monitored: PowerView, BloodHound, SharpView

**Stealthy Alternatives**
- **ADModule** — *not detected* by MDE; more silent than PowerView
- **Soaphound** — LDAP-less alternative; does not connect to LDAP port (more stealthy)

### Key Concepts

**Domain Structure**
- First user of any domain: `Administrator` — always has SID ending in **500**
- Recommended to never rename the Administrator account
- Enterprise Admins appear when enumerating forest-level domains

**Local Groups**
- Listing local groups on regular machines requires **administrative privileges**
- Listing local groups on Domain Controllers (**DC**) does **not** require admin access

**Shares**
- Use **PowerHuntShares** for share discovery and analysis
- Can discover shares, sensitive files, ACLs, networks, and identities
- Generates HTML reports for analysis

---

## 2. Domain Enumeration — Lab Practice

### Tool Setup

**Invisi-Shell (for stealthy execution)**
```powershell
# Launch Invisi-Shell with admin privileges
RunWithPathAsAdmin.bat

# Launch Invisi-Shell without admin privileges
RunWithRegistryNonAdmin.bat

# Exit and clean up when done
exit
```

**Loading AD Module**
```powershell
Import-Module C:\AD\Tools\ADModule-master\Microsoft.ActiveDirectory.Management.dll
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1
```

**Loading PowerView**
```powershell
. C:\AD\Tools\PowerView.ps1
```

---

### Domain Information

| Task | PowerView | AD Module |
|---|---|---|
| Get current domain | `Get-Domain` | `Get-ADDomain` |
| Get domain object (specific) | `Get-Domain -Domain moneycorp.local` | `Get-ADDomain -Identity moneycorp.local` |
| Get domain SID | `Get-DomainSID` | `(Get-ADDomain).DomainSID` |
| Get domain policy | `Get-DomainPolicyData` | `(Get-DomainPolicyData).systemaccess` |
| Get domain policy (specific) | `Get-DomainPolicyData -domain moneycorp.local` | `(Get-DomainPolicyData -domain moneycorp.local).systemaccess` |

**Domain Controllers**
```powershell
# Current domain
Get-DomainController                    # PowerView
Get-ADDomainController                  # AD Module

# Specific domain
Get-DomainController -Domain moneycorp.local              # PowerView
Get-ADDomainController -DomainName moneycorp.local -Discover   # AD Module
```

---

### User Enumeration

**List All Users**
```powershell
Get-DomainUser                              # PowerView
Get-DomainUser -Identity student1           # Specific user
Get-ADUser -Filter * -Properties *          # All users, all properties
Get-ADUser -Identity student1 -Properties * # Specific user, all properties
```

**User Properties & Filtering**
```powershell
# Get specific properties
Get-DomainUser -Properties samaccountname,logonCount

# Get all property names for users
Get-ADUser -Filter * -Properties * | select -First 1 | Get-Member -MemberType *Property | select Name

# Get password last set (converted to readable format)
Get-ADUser -Filter * -Properties * | select name,logoncount,@{expression={[datetime]::fromFileTime($_.pwdlastset)}}

# Search for string in user description
Get-DomainUser -LDAPFilter "Description=*built*" | Select name,Description
Get-ADUser -Filter 'Description -like "*built*"' -Properties Description | select name,Description
```

---

### Computer Enumeration

**List All Computers**
```powershell
Get-DomainComputer | select Name                       # PowerView
Get-DomainComputer -OperatingSystem "*Server 2022*"    # Filter by OS
Get-DomainComputer -Ping                               # Online check
Get-ADComputer -Filter * | select Name                 # AD Module
Get-ADComputer -Filter * -Properties *                 # All properties
```

**Filter by Operating System**
```powershell
Get-ADComputer -Filter 'OperatingSystem -like "*Server 2022*"' -Properties OperatingSystem | select Name,OperatingSystem
```

**Test Connectivity**
```powershell
Get-ADComputer -Filter * -Properties DNSHostName | %{Test-Connection -Count 1 -ComputerName $_.DNSHostName}
```

---

### Group Enumeration

**List All Groups**
```powershell
Get-DomainGroup | select Name                    # PowerView
Get-DomainGroup -Domain <targetdomain>           # Specific domain
Get-ADGroup -Filter * | select Name              # AD Module
Get-ADGroup -Filter * -Properties *              # All properties
```

**Filter by Group Name**
```powershell
Get-DomainGroup *admin*                                              # PowerView
Get-ADGroup -Filter 'Name -like "*admin*"' | select Name             # AD Module
```

**Group Membership**
```powershell
# All members of Domain Admins (including nested groups)
Get-DomainGroupMember -Identity "Domain Admins" -Recurse             # PowerView
Get-ADGroupMember -Identity "Domain Admins" -Recursive               # AD Module

# Groups a user belongs to
Get-DomainGroup -UserName "student1"                                 # PowerView
Get-ADPrincipalGroupMembership -Identity student1                    # AD Module
```

---

### Local Groups (on Machines)

> **Note:** Requires administrator privileges on non-DC machines. No admin required on Domain Controllers.

```powershell
# List local groups on a machine
Get-NetLocalGroup -ComputerName dcorp-dc

# Get members of specific local group
Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators
```

---

### Logged-On Users

**Active Sessions (requires local admin on target)**
```powershell
Get-NetLoggedon -ComputerName dcorp-adminsrv
```

**Locally Logged Users (requires remote registry)**
```powershell
Get-LoggedonLocal -ComputerName dcorp-adminsrv
```

**Last Logged User (requires admin + remote registry)**
```powershell
Get-LastLoggedOn -ComputerName dcorp-adminsrv
```

---

### Share & File Enumeration

**Find Shares**
```powershell
Invoke-ShareFinder -Verbose
Get-NetFileServer
```

**Find Sensitive Files**
```powershell
Invoke-FileFinder -Verbose
```

**PowerHuntShares (Comprehensive Share Analysis)**
```powershell
# Discover shares, files, ACLs, and generate HTML report
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```

---

## 3. BloodHound Collection

### BloodHound Legacy

**Collect Data**
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\BloodHound-master\Collectors\SharpHound.exe -args --collectionmethods All
```

---

### BloodHound CE (Current Edition)

**Collect Data**
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\Sharphound\SharpHound.exe -args --collectionmethods All
```

**Stealthy Collection (Exclude Noisy Methods)**
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SharpHound\SharpHound.exe -args --collectionmethods Group,GPOLocalGroup,Session,Trusts,ACL,Container,ObjectProps,SPNTargets,CertServices --excludedcs
```

> **Note:** Removing noisy collection methods like RDP, DCOM, PSRemote, and LocalAdmin makes BloodHound collection less conspicuous.

**Analyze Data**
- Collector gathers JSON-format files
- Upload JSON to BloodHound CE analyzer for graph visualization
- Use Cypher Queries in the BloodHound Query Library (https://queries.specterops.io)
- Lab environment provides read-only access to web UI

---

## 4. Lab Environment Overview

**Target Environment: Moneycorp (Fictional Financial Services)**
- Fully patched Server 2022 machines
- Windows Defender enabled on all hosts
- Server 2016 Forest Functional Level
- Multiple forests and multiple domains
- Minimal firewall — focus on AD attack concepts, not network evasion

---

## 5. Command Reference Cheat Sheet

| Task | Command |
|---|---|
| Load AD Module | `Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1` |
| Load PowerView | `. C:\AD\Tools\PowerView.ps1` |
| Current domain | `Get-Domain` (PV) / `Get-ADDomain` (AM) |
| All users | `Get-DomainUser` (PV) / `Get-ADUser -Filter *` (AM) |
| All computers | `Get-DomainComputer` (PV) / `Get-ADComputer -Filter *` (AM) |
| All groups | `Get-DomainGroup` (PV) / `Get-ADGroup -Filter *` (AM) |
| Group members | `Get-DomainGroupMember -Identity "Domain Admins" -Recurse` |
| Local groups on DC | `Get-NetLocalGroup -ComputerName dcorp-dc` |
| Find shares | `Invoke-ShareFinder -Verbose` |
| BloodHound (stealthy) | `C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SharpHound\SharpHound.exe -args --collectionmethods Group,GPOLocalGroup,Session,Trusts,ACL,Container,ObjectProps,SPNTargets,CertServices --excludedcs` |

---

## 6. Key Takeaways & Exam Focus

- [ ] Practice PowerView vs AD Module commands — know when each is appropriate
- [ ] ADModule is stealthier; prefer it over PowerView for real assessments
- [ ] BloodHound stealthy collection removes RDP, DCOM, PSRemote, LocalAdmin
- [ ] Local group enumeration on DC requires no admin — exploit this for reconnaissance
- [ ] Share enumeration often reveals sensitive files and misconfigured ACLs
- [ ] Always document enumeration commands and results for exam report
- [ ] Soaphound as alternative to BloodHound for LDAP-evasion scenarios

---

## Personal Notes & Questions

- [ ] Test PowerHuntShares on a larger share set in the lab
- [ ] Compare Soaphound output vs standard BloodHound for stealth differences
- [ ] Practice BloodHound Cypher queries to find attack paths manually
- [ ] Document enumeration output as it would appear in a penetration test report

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [AD Module](https://github.com/samratashok/ADModule)
- [PowerHuntShares](https://github.com/NetSPI/PowerHuntShares)
- [BloodHound CE](https://github.com/SpecterOps/BloodHound)
- [BloodHound Query Library](https://queries.specterops.io)
- [SharpHound Collector](https://github.com/SpecterOps/SharpHound)

---

*Next: Module 3 — Kerberoasting & Privilege Escalation*
