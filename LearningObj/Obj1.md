#  Learning Objective 1 — Domain Enumeration

**Date:** 25 July 2026

> Hands-on walkthrough of the first CRTP learning objective: enumerating users, computers, admin groups, and identifying write-accessible shares using PowerView, AD Module, PowerHuntShares, and BloodHound.

---

## Learning Objective 1: Domain Enumeration & Privilege Discovery

### Objective Overview
This learning objective focuses on reconnaissance and information gathering within an Active Directory environment. The goal is to:
- Enumerate domain users and computers
- Identify privileged group memberships (Domain Admins, Enterprise Admins)
- Use graph analysis (BloodHound) to find attack paths
- Discover file shares with write access — potential persistence/escalation vectors

---

## Task 1: Enumerate Domain Users

**Purpose:** Identify all user accounts in the domain, along with properties that reveal privilege levels, last logon times, and descriptions that might indicate service accounts.

### PowerView Method
```powershell
Get-DomainUser
```

**Output includes:** name, samaccountname, displayname, objectSID, pwdlastset, logoncount, and more.

### AD Module Method
```powershell
Get-ADUser -Filter * -Properties *
```

**Output includes:** All user properties (same data, different tool).

### Practical Steps (Lab)
1. Open PowerShell (via Invisi-Shell for stealth)
2. Load the tool: `. C:\AD\Tools\PowerView.ps1` OR `Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1`
3. Run enumeration command
4. Pipe to `Select` to extract specific properties:
```powershell
Get-DomainUser | Select name, samaccountname, logoncount
```

### Expected Findings
- Look for service accounts (often have descriptions like "SQL Service", "Exchange", etc.)
- Note users with high logoncount (active accounts)
- Identify accounts with old pwdlastset (possible stale/weak passwords)

---

## Task 2: Enumerate Domain Computers

**Purpose:** Identify all computer objects in the domain, filter by operating system, and determine which hosts are online.

### PowerView Method
```powershell
Get-DomainComputer | Select Name
```

**With filtering:**
```powershell
Get-DomainComputer -OperatingSystem "*Server 2022*"
Get-DomainComputer -Ping
```

### AD Module Method
```powershell
Get-ADComputer -Filter * | Select Name
Get-ADComputer -Filter 'OperatingSystem -like "*Server 2022*"' -Properties OperatingSystem | Select Name, OperatingSystem
Get-ADComputer -Filter * -Properties DNSHostName | %{Test-Connection -Count 1 -ComputerName $_.DNSHostName}
```

### Practical Steps (Lab)
1. Load PowerView or AD Module
2. Enumerate computers:
```powershell
Get-DomainComputer | Select Name, OperatingSystem
```
3. Filter for specific OS versions (targets with fewer patches are lower-hanging fruit):
```powershell
Get-DomainComputer -OperatingSystem "*2016*"
```
4. Test connectivity:
```powershell
Get-DomainComputer -Ping
```

### Expected Findings
- List of all computers in domain
- Operating systems and patch levels
- Online/offline status
- Identify older OS versions as potential targets

---

## Task 3: Enumerate Domain Admins

**Purpose:** Identify all members of the Domain Admins group (including nested group memberships). This reveals who has administrative control over the domain.

### PowerView Method
```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```

### AD Module Method
```powershell
Get-ADGroupMember -Identity "Domain Admins" -Recursive
```

### Practical Steps (Lab)
1. Load PowerView or AD Module
2. Enumerate Domain Admins:
```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```
3. Extract detailed info:
```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse | Select name, objectclass, samaccountname
```

### Expected Findings
- Direct members (users with DA rights)
- Indirect members (groups nested within Domain Admins)
- Service accounts with domain-wide admin rights
- Potential lateral movement targets

---

## Task 4: Enumerate Enterprise Admins

**Purpose:** Identify members of Enterprise Admins — a forest-level privilege group with authority across all domains.

### PowerView Method
```powershell
Get-DomainGroupMember -Identity "Enterprise Admins" -Domain moneycorp.local
```

> **Note:** Enterprise Admins exists at the forest root domain level. Query the root domain explicitly.

### AD Module Method
```powershell
Get-ADGroupMember -Identity "Enterprise Admins" -Recursive
```

### Practical Steps (Lab)
1. Load PowerView
2. Query Enterprise Admins in the forest root:
```powershell
Get-DomainGroupMember -Identity "Enterprise Admins" -Domain moneycorp.local
```
3. Compare with Domain Admins — EA members often overlap with forest-root DA, but may differ in child domains

### Expected Findings
- Forest-level administrative accounts
- Usually fewer members than domain-level admins
- Critical targets for persistent access/escalation

---

## Task 5: Use BloodHound to Identify Shortest Path to Domain Admin

**Purpose:** Leverage graph analysis to visualize attack paths and identify the shortest logical path to compromise Domain Admin privileges.

### Prerequisites
- BloodHound CE (or Legacy) running
- SharpHound data collected and uploaded

### Practical Steps (Lab)

#### 1. Open BloodHound Web UI
- Navigate to BloodHound CE interface
- Confirm SharpHound data is uploaded (via Loader):
```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SharpHound\SharpHound.exe -args --collectionmethods All
```

#### 2. Access Pre-built Searches
```
BloodHound Interface → Query Library (top menu) 
  → Pre-built Searches → Search for "Shortest Path to Domain Admin"
```

#### 3. Select Target Domain
- Specify the target domain (e.g., "dollarcorp.local")
- BloodHound calculates attack chains

#### 4. Analyze Results
- Red edges = dangerous attack paths (owned nodes → target)
- Click nodes/edges for details on how each link works
- Document the shortest path for your exam report

### Example Attack Chain (Typical)
```
Student VM (compromised)
  → Kerberoasting Service Account
    → Crack password → Service Admin privileges
      → Lateral move to domain controller
        → Dump NTDS.dit → Domain Admin hashes
```

### Expected Findings
- Shortest logical path from your current position to DA
- Multiple alternative paths (if available)
- Nodes with "Owned" status (machines/users you control)
- Visualization of ACL abuse, group nesting, and trust relationships

---

## Task 6: Find File Shares with Write Permissions

**Purpose:** Identify network shares where the current user (studentx) has write access — potential vectors for persistence, lateral movement, or privilege escalation.

### Tool: PowerHuntShares
PowerHuntShares is a stealthy alternative to `Invoke-ShareFinder` that discovers shares, analyzes ACLs, and generates an HTML report.

### Setup & Execution

#### 1. Prepare Server List
Create a text file with server hostnames/IPs:
```
C:\AD\Tools\servers.txt
```

Example content:
```
dcorp-dc
dcorp-adminsrv
dcorp-appsrv
dcorp-sql1
```

#### 2. Load PowerHuntShares Module
```powershell
Import-Module C:\AD\Tools\PowerHuntShares.psm1
```

#### 3. Run Share Hunt
```powershell
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```

**Parameters:**
- `-NoPing` — skip ping test (faster, assumes hosts are online)
- `-OutputDirectory` — where to save HTML report
- `-HostList` — text file containing server names/IPs

### Output
- **HTML Report** — saved in OutputDirectory with findings
- **Console output** — lists discovered shares, ACLs, sensitive files
- **Sensitive file detection** — flags files with keywords (password, config, key, etc.)

### Practical Steps (Lab)
1. Create servers.txt with domain hosts
2. Load PowerHuntShares module
3. Run scan:
```powershell
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```
4. Open generated HTML report in browser
5. Filter shares by access level (read, write)
6. Identify shares where `studentx` has write permissions

### Expected Findings
- SYSVOL, NETLOGON (always readable by domain users)
- Application shares with overly permissive ACLs
- Backup shares with write access (data exfiltration or persistence)
- Home directories (personal share for current user)
- Shares hosting scripts or executables (potential execution)

### Example Write-Accessible Share Scenario
```
Share: \\dcorp-appsrv\deploy
Owner: DCORP\AppAdmins
Permissions: DCORP\Domain Users (Write)
Sensitive Files: app-config.xml, db-password.txt
Risk: Modify application configuration or inject malicious code
```

---

## Summary of Commands Used

| Task | Command |
|---|---|
| Enumerate users | `Get-DomainUser` (PV) / `Get-ADUser -Filter * -Properties *` (AM) |
| Enumerate computers | `Get-DomainComputer \| Select Name` (PV) / `Get-ADComputer -Filter *` (AM) |
| Domain Admins | `Get-DomainGroupMember -Identity "Domain Admins" -Recurse` (PV) / `Get-ADGroupMember -Identity "Domain Admins" -Recursive` (AM) |
| Enterprise Admins | `Get-DomainGroupMember -Identity "Enterprise Admins" -Domain moneycorp.local` (PV) |
| BloodHound Analysis | Query Library → Pre-built Searches → Shortest Path to Domain Admin |
| Find write-accessible shares | `Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt` |

---

## Report Writing Notes

For your CRTP exam report, include:

**1. Enumeration Summary**
```
Discovered X users, Y computers, Z groups in the domain.
Domain Admins: [List members]
Enterprise Admins: [List members]
Identified [N] shares with write access for studentx account.
```

**2. BloodHound Path Visualization**
- Screenshot of attack chain
- Explanation of each step
- Tools required for each hop

**3. Share Access Summary**
- Table of writable shares
- Risk assessment (what can be done with write access)
- Potential exploitation scenarios

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [AD Module](https://github.com/samratashok/ADModule)
- [PowerHuntShares](https://github.com/NetSPI/PowerHuntShares)
- [BloodHound CE](https://github.com/SpecterOps/BloodHound)
- [BloodHound Query Library](https://queries.specterops.io)

---

*Next: Learning Objective 2 — Kerberoasting & AS-REP Roasting*
