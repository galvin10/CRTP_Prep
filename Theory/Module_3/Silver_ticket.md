# Silver Ticket Attack

**Date:** 03 August 2026


---

## What is a Silver Ticket?

A forged **Service Ticket (ST)** created using the **target service account's hash**.

Unlike Golden Tickets (TGT), Silver Tickets:
- Only provide access to **specific service**
- Use machine account hash (rotates every 30 days)
- **No connection to KDC** needed
- **Not detected by MDI** (Microsoft Defender for Identity)
- Extremely stealthy

---

## Attack Steps

**Step 1: Acquire Target Service Account Hash**
- Extract AES256 key or NTLM hash
- For Windows services: use machine account hash
- Source: LSASS dump, DCSync, or memory

**Step 2: Forge Service Ticket**
- Use Rubeus or similar tool
- Specify target service (HTTP, CIFS, HOST, etc.)
- Forge ST using service account hash
- Inject into memory (/ptt)

**Step 3: Access Service**
- Use forged ticket to access service
- Service validates ticket (doesn't contact KDC)
- Access granted without DC involvement

---

## Prerequisites

**For Windows Services:**
- Services run as **machine account** by default
- Must acquire **machine account hash**
- Rotates every 30 days (limits persistence)

**Service Accounts:**
- Can target domain service accounts
- Need their AES256 or NTLM hash
- Persistence depends on password change frequency

---

## Why Silver Tickets Are Stealthy

✅ **No DC connection** — service doesn't validate with KDC  
✅ **MDI blind** — Microsoft Defender for Identity doesn't detect  
✅ **No TGT creation** — only forged ST (service-specific)  
✅ **Silent** — minimal network noise  
✅ **Optional PAC validation only** — most services don't enable it  
✅ **Works for any service** — HOST, CIFS, HTTP, RPCSS, etc.

---

## Silver Ticket vs Golden Ticket

| Aspect | Silver Ticket | Golden Ticket |
|---|---|---|
| What's forged | Service Ticket (ST) | TGT |
| Hash used | Service account hash | KRBTGT hash |
| Scope | Single service | Entire domain |
| Persistence | 30 days (account rotation) | 10+ hours (TGT lifetime) |
| DC connection | No | No (but more suspicious) |
| MDI detection | No | Yes (usually) |
| Noise level | Very silent | Noisy |
| OPSEC | Excellent | Good with evasion |

---

## Forge Silver Ticket

**Basic command:**

```powershell
C:\AD\Tools\Rubeus.exe silver 
  /service:http/dcorp-dc.dollarcorp.moneycorp.local 
  /rc4:6e58e06e07588123319fe02feeab775d 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ldap 
  /user:Administrator
  /domain:dollarcorp.moneycorp.local 
  /ptt
```

**Parameters:**
- `/service` — Target service (format: service/hostname.domain)
- `/rc4` — NTLM hash (RC4) of service account
- `/aes256` — AES256 key (alternative to /rc4)
- `/sid` — Domain SID
- `/ldap` — Query DC for user info (convenience; use manual values for OPSEC)
- `/user` — User to impersonate (Administrator)
- `/domain` — Domain FQDN
- `/ptt` — Pass-the-ticket (load into memory)

---

## Common Service SPNs

Target any of these services on a machine:

| Service | SPN Format |
|---|---|
| HTTP/HTTPS | `http/server.domain.local` |
| SMB/CIFS | `cifs/server.domain.local` |
| RPC | `rpcss/server.domain.local` |
| HOST | `host/server.domain.local` |
| LDAP | `ldap/server.domain.local` |
| SQL Server | `mssqlsvc/server.domain.local` |

---

## Practical Examples

**HTTP Service on Domain Controller:**
```powershell
C:\AD\Tools\Rubeus.exe silver 
  /service:http/dcorp-dc.dollarcorp.moneycorp.local 
  /rc4:6e58e06e07588123319fe02feeab775d 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

**CIFS Service (SMB/File Access):**
```powershell
C:\AD\Tools\Rubeus.exe silver 
  /service:cifs/dcorp-dc.dollarcorp.moneycorp.local 
  /aes256:<aes256_key> 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

---

## Detection Bypass

**PAC Validation (Optional):**
- Most services don't enable optional PAC validation
- Even if enabled, Silver Ticket may still work
- Depends on service implementation

**No Kerberos Pre-authentication:**
- Silver Ticket doesn't require KDC
- Completely offline attack
- MDI/XDR cannot detect KDC requests

---

## Limitations

❌ **Service-specific** — only access that one service  
❌ **Account rotation** — 30-day persistence (for machine accounts)  
❌ **One service per ticket** — need new ticket for different service  
❌ **May need multiple tickets** — for multi-service access  

---

## Detection Considerations

**What won't detect:**
- MDI (Microsoft Defender for Identity)
- KDC audit logs (no KDC involvement)
- Network monitoring (minimal traffic)

**What might detect:**
- Event ID 4624 (Logon event — unusual if paired with service access)
- PAC validation (if enabled on service)
- Behavioral anomalies (e.g., admin access from non-admin context)
- Microsoft XDR (endpoint detection & response)

---

## Best Use Cases

1. **Persistence** — machine account rotates in 30 days (longer than TGT)
2. **Stealth** — no KDC involvement, MDI-safe
3. **Lateral movement** — specific service access without domain admin
4. **Targeted attacks** — access only needed service (minimizes exposure)

---

## References

- [Rubeus - Silver Ticket](https://github.com/GhostPack/Rubeus)
- [Active Directory Kerberos - Microsoft](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)

---

*Next: Learning Objective 9 — Cross-Forest Attacks*
