# Privilege Escalation — GPO Abuse

**Date:** 27 July 2026

---

## GPO Abuse Overview

A GPO with overly permissive ACL can be abused for multiple attacks.

**Technique: GPOddity**
- Combines NTLM relaying and modification of Group Policy Container
- Relay credentials of user who has WriteDACL on GPO
- Modify gPCFileSysPath (group policy template path, default is SYSVOL)
- Load malicious template from attacker-controlled location

> **Note:** Very noisy but very powerful and useful

---

## Identify GPOs with WriteDACL Access

**Using PowerView:**
```powershell
Get-DomainGPO
```

Look for `gPCFileSysPath` attribute in output. This shows the path where GPO templates are stored.

**Filter for interesting GPOs:**
```powershell
Get-DomainGPO | Select displayname, gPCFileSysPath
```

---

## Attack Steps (GPOddity)

1. **Identify GPO with WriteDACL permission**
   - Current user must have WriteDACL on target GPO
   - Use ACL enumeration from Learning Objective 2

2. **Setup NTLM relay**
   - Use Responder or similar tool to capture NTLM credentials
   - Relay credentials to LDAP/ADCS

3. **Modify gPCFileSysPath**
   - After relay, modify the GPO's file system path
   - Point to attacker-controlled UNC path

4. **Place Malicious Template**
   - Host malicious GPO template on attacker's SMB share
   - Template executes when GPO is applied (next group policy refresh)

5. **Trigger Group Policy Update**
   - Force `gpupdate /force` on affected machines
   - Or wait for scheduled group policy refresh (every 90 minutes)

---

## Detection & Impact

**Detection:**
- Very noisy in logs (GPO modifications, unusual file paths)
- Monitor group policy application events

**Impact:**
- Arbitrary code execution as SYSTEM on all machines where GPO applies
- Persistence across reboots
- Affects all users logging into affected machines

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [GPOddity (NTLM Relay + GPO Abuse)](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-gpo-permissions)

---

*Next: Learning Objective 6 — Credential Extraction & Lateral Movement*
