# Lateral Movement — Credential Extraction

**Date:** 27 July 2026


---

## LSASS Overview

**Local Security Authority Subsystem Service (LSASS):**
- Responsible for Windows authentication
- Stores credentials in multiple forms: NT hash, AES, Kerberos tickets
- Most attractive target for credential extraction
- Most monitored process on Windows

---

## When Credentials Are Stored in LSASS

- User logs on to local session or RDP
- RunAs execution
- Windows service execution
- Scheduled task or batch job execution
- Remote administration tool usage

---

## Credential Extraction Without Touching LSASS

**SAM Hive (Registry)**
- Local user credentials
- Registry path: `HKEY_LOCAL_MACHINE\SAM`
- Extract using tools like `reg save` or `mimikatz`

**LSA Secrets/SECURITY Hive (Registry)**
- Service account passwords
- Domain cached credentials
- Registry path: `HKEY_LOCAL_MACHINE\SECURITY`

**DPAPI Protected Credentials (Disk)**
- Credentials Manager/Vault
- Browser cookies
- Certificates
- Azure tokens
- Requires DPAPI master key

**Insecure Storage**
- Credentials in file shares
- Batch scripts with hardcoded passwords
- Console history (.PowerShell_history, .bash_history)
- Log files

**Social Engineering**
- Tricking users
- Phishing
- Consent grants
- Credential reuse

---

## Extraction Methods

| Source | Tool | Stealth |
|---|---|---|
| SAM + SYSTEM hive | mimikatz, Invoke-SAMRUDump | Medium |
| LSA Secrets | secretsdump.py, Invoke-LSASecrets | Medium |
| DPAPI | dpapi.py, SharpDPAPI | Low |
| File shares | dir, findstr, grep | High |
| Console history | type `$PROFILE`, cat ~/.bash_history | High |

---

## References

- [Mimikatz](https://github.com/gentilkiwi/mimikatz)
- [Impacket secretsdump](https://github.com/fortra/impacket)
- [SharpDPAPI](https://github.com/GhostPack/SharpDPAPI)

---

*Next: Learning Objective 7 — Kerberoasting & Lateral Movement*
