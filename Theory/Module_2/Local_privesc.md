# Privilege Escalation — Local & Domain Escalation Techniques

**Date:** 26 July 2026

---

## Privilege Escalation Scenarios

**Attack vectors:**
- Hunting for local admin access on other machines
- Hunting for high-privilege domain accounts (Domain Admins)
- Local privilege escalation on Windows

---

## Local Privilege Escalation Methods

- Missing patches
- Automated deployment and AutoLogon passwords in clear text
- AlwaysInstallElevated (Any user can run MSI as SYSTEM)
- Misconfigured services
- DLL hijacking
- Kerberos and NTLM relaying

---

## Privilege Escalation Tools

```powershell
# PowerUp
https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc

# PrivescCheck
https://github.com/itm4n/PrivescCheck

# winPEAS
https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS
```

---

## Feature Abuse

Relying on feature abuse rather than patch-based exploits. Features are rarely patched and not security team's focus.

**Example: Jenkins**
- Jenkins is a widely used CI tool
- Default installations (pre-2.x) allow command execution
- Often runs with SYSTEM or admin privileges

**Exploitation:**
1. **With admin access:** Navigate to `http://<jenkins_server>/script` → execute Groovy scripts
2. **Without admin access:** Add build steps → "Execute Windows Batch Command" → run:
```powershell
powershell -c <command>
```

Can download/execute scripts, run encoded scripts, etc.

---

## Privilege Escalation — Relaying

In a relaying attack, credentials are not captured but forwarded to a local/remote service for authentication.

**Types:**
- **NTLM relaying**
- **Kerberos relaying**

**Most abused services:** LDAP and AD CS

---

## Local Service Issues (PowerUp)

**Unquoted service paths with spaces:**
```powershell
Get-ServiceUnquoted -Verbose
```

**Services where current user can modify binary path:**
```powershell
Get-ModifiableServiceFile -Verbose
```

**Services with modifiable configuration:**
```powershell
Get-ModifiableService -Verbose
```

---

## Run All Privilege Escalation Checks

```powershell
# PowerUp
Invoke-AllChecks

# PrivescCheck
Invoke-PrivEscCheck

# winPEAS
winPEASx64.exe
```

---

## References

- [PowerUp](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc)
- [PrivescCheck](https://github.com/itm4n/PrivescCheck)
- [winPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS)

---

*Next: Learning Objective 5 — Kerberoasting & Credential Extraction*
