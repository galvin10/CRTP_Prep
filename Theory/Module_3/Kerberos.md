#  Kerberos Authentication

**Date:** 01 August 2026

---

## What is Kerberos?

Kerberos is a **ticket-based authentication protocol** used by Active Directory.

**Simple Analogy:** Think of it like a nightclub:
- You show your ID to the bouncer (KDC)
- Bouncer gives you a wristband (TGT)
- You use the wristband to enter different areas of the club (access resources)

---

## Key Components

**1. KDC (Key Distribution Center)**
- Runs on Domain Controller
- Issues tickets
- Knows everyone's password/keys

**2. Client/User**
- Requests access to resources
- Stores tickets locally

**3. Service/Server**
- Resource you want to access (fileserver, SQL, etc.)
- Verifies tickets

---

## Simple Authentication Flow

### Step 1: Authentication (Get TGT)
```
User asks KDC: "I'm Admin, let me in"
KDC checks password: "Valid! Here's your Ticket-Granting Ticket (TGT)"
TGT = proof you are who you claim to be
```

### Step 2: Authorization (Get Service Ticket)
```
User asks KDC: "I need to access fileserver"
KDC checks TGT: "Valid! Here's your Service Ticket (ST)"
ST = permission to access specific service
```

### Step 3: Access Service
```
User gives ST to fileserver
Fileserver verifies ST: "Looks good! Access granted"
```

---

## Key Terms

**TGT (Ticket-Granting Ticket)**
- Proves you authenticated to the domain
- Obtained once at logon
- Valid for 10 hours (default)
- Can be reused to request service tickets

**ST (Service Ticket)**
- Proves you can access specific service
- Shorter lived than TGT
- Obtained on-demand when accessing resources

**SPN (Service Principal Name)**
- Unique identifier for a service
- Format: `service/hostname.domain.local`
- Example: `MSSQLSvc/sql-server.corp.local`

**Credentials Cached**
- NT hash (NTLM hash of password)
- Kerberos keys (AES256, AES128, RC4)
- Used to encrypt/decrypt tickets

---

## Visual Flow

```
┌─────────────────────────────────────────────────────┐
│                    KERBEROS FLOW                     │
└─────────────────────────────────────────────────────┘

1. USER LOGIN
   ↓
   User sends username + timestamp (encrypted with password)
   ↓
   KDC verifies & sends TGT back
   ↓
   TGT stored in memory (lsass.exe)

2. REQUEST SERVICE
   ↓
   User sends TGT + SPN to KDC
   ↓
   KDC verifies TGT & creates Service Ticket
   ↓
   ST sent back to user

3. ACCESS SERVICE
   ↓
   User sends ST to target service
   ↓
   Service verifies ST
   ↓
   Access granted
```

---

## Why Kerberos Matters for CRTP

**Kerberos is exploitable because:**

1. **Tickets are cached in memory** — extract via mimikatz
2. **SPN = predictable** — hunt for high-value services
3. **Weak encryption** — crack hashes offline
4. **No password verification** — relay attacks work
5. **Tickets can be forged** — Golden/Silver Ticket attacks

---

## Common Kerberos Attacks (Overview)

**Kerberoasting**
- Steal service ticket from memory
- Crack hash offline
- Impersonate service account

**AS-REP Roasting**
- Target users without Kerberos preauthentication
- Forge authentication request
- Crack hash offline

**Pass-the-Ticket**
- Steal TGT or ST from memory
- Use it to access resources
- No password needed

**Golden Ticket**
- Forge TGT with KRBTGT hash
- Unlimited access to domain
- Persistent backdoor

**Silver Ticket**
- Forge ST for specific service
- Access that service
- Service-specific compromise

**Relay Attacks**
- Capture NTLM/Kerberos auth
- Forward to another service
- Escalate privileges

---

## Quick Reference

| Component | Purpose | Validity |
|---|---|---|
| TGT | Prove authenticated to domain | 10 hours |
| ST | Access specific service | 10 hours |
| SPN | Identify service uniquely | N/A |
| NT Hash | Encrypt/decrypt tickets | Lifetime |
| Kerberos Key | Encrypt/decrypt tickets | Lifetime |

---

## Key Takeaway

```
Kerberos = Ticket-based authentication

No tickets in memory = No access
Tickets in memory = Can be stolen/forged/relayed

Defensive: Monitor ticket requests & memory access
Offensive: Extract, forge, or relay tickets
```

---

## In Practice (CRTP Labs)

1. **Find high-value SPNs** → `Get-DomainUser -SPN`
2. **Request service tickets** → `GetUserSPNs.py`
3. **Extract from memory** → `Invoke-Mimikatz`
4. **Crack hashes** → `hashcat`
5. **Impersonate user** → `Rubeus asktgt`

---

## References

- [Kerberos Protocol (Microsoft)](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)
- [HackTricks: Kerberos](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/kerberos-authentication)

---

*Next: Kerberoasting & AS-REP Roasting Attacks*
