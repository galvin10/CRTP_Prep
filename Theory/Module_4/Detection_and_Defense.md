# Detection and Defense — Active Directory Security & Defensive Strategies

**Date:** 20 August 2026


---

## Introduction to AD Defense

**Modern AD defense requires multi-layered approach:**
- Administrative isolation (PAWs, ESAE)
- Credential protection (Protected Users, JIT)
- Monitoring & detection (4769, 4662 logging)
- Deception techniques (honeypots, decoy objects)
- Zero Trust architecture (Privileged Access Strategy)

---

## Strategy 1: Protect and Limit Domain Admins

**Domain Admins = Highest value target for attackers**

### Reduce Domain Admin Footprint

**Best practices:**

```
1. Minimize number of Domain Admins
   - Have only essential DAs
   - Document each DA and justification
   - Remove unnecessary DAs regularly

2. Restrict DA login locations
   - DAs should only login to Domain Controllers
   - Disable login on workstations
   - Disable login on servers
   - GPO: "Deny log on locally"

3. Never run services as DA
   - Service accounts = credential theft target
   - Use managed service accounts instead
   - Separate service account from DA account
   - Use Group Managed Service Accounts (gMSA)
```

---

### Set "Account is Sensitive and Cannot be Delegated"

**Prevents credential delegation attacks:**

```powershell
# Set for all Domain Admins
Get-ADGroupMember "Domain Admins" | ForEach-Object {
  Set-ADUser -Identity $_ -AccountNotDelegated $true
}

# Verify setting
Get-ADUser -Identity DA_Account -Properties AccountNotDelegated
# Output: AccountNotDelegated = $true
```

**What this prevents:**
- Constrained delegation attacks (S4U2Self/S4U2Proxy)
- Unconstrained delegation abuse
- Kerberos delegation exploitation
- Credential reuse via delegation

---

## Strategy 2: Protected Users Group

**Protected Users = Special group for high-value accounts (Server 2012 R2+)**

### Device Protections (All versions)

**When user is member of Protected Users:**

```
✓ Cannot use CredSSP
  └─ No CredSSP authentication (no cleartext cred caching)

✓ Cannot use WDigest
  └─ No WDigest authentication (no cleartext password caching)

✓ NTLM hash NOT cached
  └─ Cannot extract via LSASS dump
  └─ Cannot use Pass-the-Hash attacks

✓ Kerberos uses AES only
  └─ No DES encryption (deprecated)
  └─ No RC4 encryption
  └─ Only strong AES-256/AES-128
  └─ No cleartext credential caching
  └─ No long-term key caching
```

---

### Domain Controller Protections (Server 2012 R2+)

**When domain functional level is 2012 R2:**

```
✓ No NTLM authentication
  └─ NTLMv2 attacks impossible
  └─ NTLM relay attacks blocked
  └─ Kerberos required

✓ No DES/RC4 in Kerberos pre-auth
  └─ Only strong encryption
  └─ AS-REP roasting becomes harder

✓ No delegation allowed
  └─ Constrained delegation disabled
  └─ Unconstrained delegation disabled
  └─ S4U attacks impossible

✓ No TGT renewal beyond 4 hours
  └─ Hardcoded (unconfigurable)
  └─ TGT expires after 4 hours
  └─ Cannot be renewed indefinitely
  └─ Reduces ticket lifetime attacks
```

---

### Requirements & Limitations

**Requirements:**

```
- All Domain Controllers must be Server 2008 or later
  (needed for AES key support)
- All client machines should support Protected Users
- Requires testing before deployment
```

---

**Limitations:**

```
✗ Not recommended for DAs/EAs without testing
  └─ Risk of account lockout
  └─ Potential service disruption
  └─ Microsoft recommends thorough testing

✗ No cached logon (no offline sign-on)
  └─ Disconnected machines cannot authenticate
  └─ Laptops will have issues when offline
  └─ Not suitable for remote/mobile users

✗ Computer accounts useless in group
  └─ Computer credentials always on disk
  └─ Group membership doesn't protect machines
  └─ Only useful for user accounts

✗ Service accounts useless in group
  └─ Service credentials always in memory
  └─ Group membership doesn't help
  └─ Use Group Managed Service Accounts instead
```

---

### Implementation

```powershell
# Add high-value user to Protected Users
Add-ADGroupMember -Identity "Protected Users" -Members user@domain

# Add multiple users
@("da1@domain", "da2@domain", "admin@domain") | 
  ForEach-Object { Add-ADGroupMember -Identity "Protected Users" -Members $_ }

# Verify membership
Get-ADGroupMember -Identity "Protected Users"
```

---

## Strategy 3: Privileged Administrative Workstations (PAWs)

**PAW = Hardened workstation for sensitive administrative tasks**

### What is a PAW?

```
Dedicated machine for:
- Domain Controller administration
- Cloud infrastructure management
- Sensitive business functions
- High-value account management
- Critical asset administration
```

---

### Protection Offered by PAW

**Protects against:**

```
✓ Phishing attacks
  └─ No user email/web browsing on PAW
  └─ Isolated from infection vectors
  └─ No credential harvesting malware

✓ OS vulnerabilities
  └─ Minimal services running
  └─ Hardened configuration
  └─ Regular patching
  └─ No unnecessary software

✓ Credential replay attacks
  └─ Credentials not exposed to malware
  └─ No LSASS dumping risk
  └─ No credential theft possible
  └─ Isolated network (separate VLAN)

✓ Lateral movement
  └─ Admin credentials isolated
  └─ Cannot spread to other machines
  └─ Separate hardware/network
```

---

### PAW Architecture

**Separate privilege and hardware:**

```
PAW Model (3-tier):

Tier 0 (PAW - Most Secure)
├─ Only Domain Controller admin
├─ Only access to DC
├─ No internet
├─ No email
├─ No general computing
└─ Hardened OS + monitoring

Tier 1 (Admin Server)
├─ Access to Tier 1 servers
├─ Limited access to Tier 0
├─ Accessed only from Tier 0
└─ Administrative servers

Tier 2 (User workstations)
├─ General computing
├─ Email/internet access
├─ Standard user privileges
└─ Separate from PAW
```

---

### PAW Implementation

**Option 1: Separate Physical Hardware**

```
- Dedicated laptop/desktop for admin tasks
- No other use on this device
- Network isolated (separate VLAN)
- Strong authentication (smartcard)
- Minimal software
- Regular patching & monitoring
```

---

**Option 2: VM-based PAW**

```
PAW with internal VM:

Physical PAW
├─ Hardened host OS
├─ Minimal services
├─ Admin tasks via RDP
└─ VM for user tasks
    ├─ Standard OS
    ├─ Email/internet
    ├─ General computing
    └─ Isolated from admin resources
```

---

### Access to Admin Jump Servers

```
User → PAW → Authenticate → Admin Jump Server → DC

Access control:
- Only from PAW (network control)
- Strong authentication (MFA)
- Separate admin account (not user account)
- Logging & monitoring enabled
- Time-limited sessions
```

---

## Strategy 4: Just In Time (JIT) Administration

**JIT = Time-bound administrative access granted on per-request basis**

### Concept

```
Instead of: User always has DA privileges
Use JIT: User requests DA access for 1 hour
         After 1 hour: Access automatically revoked
```

---

### Temporary Group Membership

**Grant time-limited group membership:**

```powershell
# Add user to Domain Admins for 60 minutes
Add-ADGroupMember -Identity 'Domain Admins' -Members newDA `
  -MemberTimeToLive (New-TimeSpan -Minutes 60)

# After 60 minutes: Automatically removed from group
# No manual removal needed
```

---

### JIT Benefits

```
✓ Principle of Least Privilege
  └─ Users only have DA when needed
  └─ Exposure window minimized
  └─ Reduced attack surface

✓ Auditing
  └─ Request timestamp logged
  └─ Who requested recorded
  └─ Purpose captured (optional)

✓ Automatic Revocation
  └─ No manual removal needed
  └─ Time-based expiration
  └─ Cannot "forget" to remove access

✓ Reduced Credential Theft Risk
  └─ Shorter window for compromise
  └─ Stolen DA creds become useless after timeout
  └─ Impact limited to time window
```

---

### Requirements

```
✓ Requires Privileged Access Management (PAM) feature
  └─ Must be enabled in AD
  └─ WARNING: Cannot be turned off later
  └─ Requires Server 2016 or later
```

---

## Strategy 5: Just Enough Administration (JEA)

**JEA = Role-based access control for PowerShell remote administration**

### Concept

```
JEA allows:
- Non-admin users to perform specific admin tasks
- Delegated administration via PowerShell
- Granular command control
- Parameter restriction
```

---

### How JEA Works

**Define JEA endpoint:**

```powershell
# Create JEA configuration
$JEAConfig = @{
    Version = '1.0.0'
    Description = 'Delegate user management'
    
    RoleDefinitions = @{
        'UserManager' = @{
            Visible = 'New-ADUser', 'Set-ADUser', 'Remove-ADUser'
            VisibleFunctions = 'Get-UserInfo'
            RestrictedFunctions = 'Format-List'
        }
    }
    
    RunAsVirtualAccount = $true
}

# Register endpoint
Register-PSSessionConfiguration -Path JEAConfig.pssc
```

---

### Access Control

**Users can only:**

```
✓ Run allowed commands
  └─ Only specified PowerShell commands
  └─ Other commands blocked
  └─ Command execution in context of admin

✓ Use restricted parameters
  └─ Only allow safe parameter values
  └─ Prevent dangerous flags
  └─ Safe delegation to non-admins

✗ Cannot
  └─ Exit restricted endpoint
  └─ Access unrestricted PowerShell
  └─ Run unauthorized commands
  └─ Use unauthorized parameters
```

---

### Logging & Transcription

**All JEA activity logged:**

```powershell
# JEA sessions have automatic logging
- PowerShell transcription enabled
- All commands recorded
- Complete audit trail
- Cannot be disabled per session

# Output files location:
C:\ProgramData\JEA\Transcripts\
```

---

### JEA Example

```powershell
# Allow user to reset passwords only

RoleDefinitions = @{
    'PasswordReset' = @{
        Visible = 'Set-ADAccountPassword'
        VisibleParameters = @{
            SetADAccountPassword = @{
                Parameters = 'Identity'
                ValidateSet = 'Standard User Accounts Only'
            }
        }
    }
}

# User can ONLY reset passwords, nothing else
```

---

## Strategy 6: ESAE (Enhanced Security Admin Environment)

**ESAE = Dedicated administrative forest (Red Forest)**

### Concept

```
Production Forest          Administrative Forest (Red Forest)
├─ User accounts          ├─ DA accounts
├─ Workstations           ├─ EA accounts
├─ Servers                ├─ Service accounts
└─ Data                   └─ Admin jump servers

Forest trust: One-way, selective authentication
Production → Red Forest (read-only)
Red Forest ← Production (denied)
```

---

### Forest as Security Boundary

```
Important concept:
- Domain = NOT a security boundary
  └─ DA in child can compromise parent
  └─ Compromise spreads forest-wide

- Forest = SECURITY BOUNDARY
  └─ Forest trust is explicit
  └─ Can be selective authentication
  └─ Requires explicit trust relationship
  └─ More defensible
```

---

### ESAE Architecture

```
Production Forest (Tier 2 & 3)
└─ Normal users & servers
   └─ May be compromised

├─ One-way trust to Red Forest
└─ Selective authentication

Red Forest (Tier 0 - Secure)
├─ Administrative users
├─ Domain Controllers
├─ Admin jump servers
└─ Critical resources only
   └─ NOT compromised by production breach
```

---

### Administrative User Isolation

**In Red Forest:**

```
Admin user @production.local
└─ Created as @redforest.local
└─ Used as STANDARD non-privileged user
└─ Only elevated for specific admin tasks
└─ Separate admin account for production access
```

---

### Selective Authentication

**Control who can authenticate to Red Forest:**

```
Without Selective Auth:
- Any user from production can auth to Red Forest
- Risky if production compromised

With Selective Auth:
- Only authorized users can auth
- Group-based access control
- Explicit allow-list
- Denies by default
```

**Configuration:**

```powershell
# Enable selective authentication on forest trust
Set-ADForestTrust -Identity redforest.local `
  -SelectiveAuthenticationEnabled $true

# Add allowed users group
Set-ADForestTrust -Identity redforest.local `
  -AllowedUsers 'Production\Tier0Admins'
```

---

### ESAE Benefits

```
✓ Isolated admin credentials
  └─ Not exposed to production compromise
  └─ Cannot be stolen from workstations
  └─ Protected from lateral movement

✓ Hardened infrastructure
  └─ Minimal services
  └─ Regular patching
  └─ Strong monitoring
  └─ No user data

✓ Reduced attack surface
  └─ Fewer targets
  └─ More defensible
  └─ Easier to monitor
  └─ Simpler to harden

✓ Compliance
  └─ Meets regulatory requirements
  └─ Separation of duties
  └─ Admin isolation
  └─ Audit trail
```

---

### ESAE Limitations

```
✗ Complex to implement
  └─ Requires forest infrastructure
  └─ Additional DCs needed
  └─ Trust relationships to manage
  └─ User account duplication

✗ Cost
  └─ Separate forest infrastructure
  └─ Additional hardware
  └─ Maintenance overhead
  └─ Personnel training

✗ User experience
  └─ Multiple accounts needed
  └─ More complex processes
  └─ Additional authentication steps
```

---

### Microsoft Retirement (2021)

**Microsoft replaced ESAE with:**
- Privileged Access Strategy
- Zero Trust architecture
- Cloud-based solutions (Azure)
- Modern identity management

**But ESAE still valid for:**
- On-premises only environments
- High-security requirements
- Compliance-heavy organizations

---

## Strategy 7: Privileged Access Strategy (Modern)

**Microsoft's guidance for modern AD security (replaces ESAE)**

### Core Principles

```
Zero Trust Architecture:
1. Verify explicitly
   └─ Never trust, always verify
   └─ Strong authentication (MFA)
   └─ Device compliance checking

2. Use least privilege access
   └─ Only necessary permissions
   └─ Time-limited access (JIT)
   └─ Role-based access (JEA)

3. Assume breach
   └─ Design for breach
   └─ Rapid detection & response
   └─ Isolated credentials
   └─ Limited blast radius
```

---

### Three Planes Model (replaces Tier Model)

```
Control Plane
├─ Access control mechanisms
├─ Identity management
├─ Authentication & MFA
├─ Authorization policies
└─ Primary control: Identity

Management Plane
├─ Asset management
├─ Monitoring & logging
├─ Configuration management
├─ Compliance checking
└─ Observability: All activities

Data/Workload Plane
├─ Business-value assets
├─ Applications & data
├─ Workloads & services
├─ IP & intellectual property
└─ Protect: Core business
```

---

### Access Vectors

**User access:**
- Employee access (on corporate network)
- Public access (internet users)
- B2B access (partner organizations)

**App access:**
- API access (service-to-service)
- Application authentication
- Workload identity

---

### Cloud-Centric Approach

```
"Cloud is a source of security"

Leverage:
- Azure AD (cloud identity)
- Azure services (hardened)
- Cloud-based PAM
- Managed services
- Zero Trust network (Zero Trust Edge)

Modern approach:
- Reduce on-premises complexity
- Offload to cloud
- Managed security services
- Faster updates & patches
- Better visibility
```

---

### Rapid Modernization Plan (RAMP)

**Microsoft's guidance for implementation:**

```
1. Assess current state
2. Develop strategy
3. Prioritize initiatives
4. Implement in phases
5. Monitor & adjust
6. Continuous improvement
```

---

## Strategy 8: Deception & Honeypots

**Use decoy objects to detect attackers**

### Decoy Users

**Create fake high-value accounts:**

```powershell
# Create decoy Domain Admin user
Create-DecoyUser -UserFirstName admin -UserLastName manager `
  -Password Pass@123 | 
  Deploy-UserDeception -UserFlag PasswordNeverExpires `
  -GUID d07da11f-8a3d-42b6-b0aa-76c962be719a -Verbose
```

---

### Special LDAP Property Monitoring

**Set 4662 logging on special attributes:**

```
Attribute: x500uniqueIdentifier
GUID: d07da11f-8a3d-42b6-b0aa-76c962be719a

Trigger event 4662 whenever:
- PowerView reads this attribute
- ADExplorer reads this attribute
- Mimikatz enumerates this attribute

Does NOT trigger on:
- net.exe queries
- WMI classes (Win32_UserAccount)
- ActiveDirectory PowerShell module
```

---

### Why Attackers Read This Attribute

**Attackers read user objects to find:**

```
✓ High-privilege users
  └─ Domain Admins
  └─ Enterprise Admins
  └─ Service accounts

✓ Sensitive attributes
  └─ Permissions over objects
  └─ Group memberships
  └─ Delegation settings

✓ Misconfigured objects
  └─ Dangerous ACLs
  └─ Bad attribute settings
  └─ SIDHistory
  └─ TrustedToAuth flag
```

---

### Deploy-Deception Tool

**GitHub:** https://github.com/samratashok/Deploy-Deception

**Features:**

```
- Create decoy users
- Set honeypot attributes
- Trigger logging on access
- Multiple deception strategies
- Integration with monitoring
```

---

### Deception Strategy

**What to deceive:**

```
Create decoy objects with:
- High privileges (DA flag)
- Dangerous permissions (DCSync rights)
- Suspicious attributes (TrustedToAuth)
- SID History injection
- Unconstrained delegation

Bait attackers to:
- Enumerate deception objects
- Try to access them
- Attempt to steal credentials
- Trigger 4662 alerts
```

---

### Benefits

```
✓ Detection opportunity
  └─ Any access = attacker activity
  └─ High confidence alert
  └─ Impossible for defender to trigger

✓ Increase attacker cost
  └─ Time spent on fake objects
  └─ Wrong attack paths followed
  └─ Slows down progression

✓ Early detection
  └─ Detects during enumeration phase
  └─ Before privilege escalation
  └─ Before lateral movement
  └─ Earlier intervention
```

---

## Strategy 9: Kerberos Ticket Logging & Monitoring

**Monitor Kerberos activity (Event 4769)**

### Event 4769: Kerberos Service Ticket Request

**Logged on:** Domain Controller

**Frequency:** VERY HIGH (thousands per day)

```
Example log:
- Service: ldap/DC.domain.local
- User: domain\user
- Result: Success (0x0)
- Encryption: 0x17 (AES-256)
- Ticket flags: 0x40800000
```

---

### Filtering 4769 Logs

**Filter out noise:**

```
Keep only suspicious:

✓ Service name should NOT be krbtgt
  └─ krbtgt = ticket renewal
  └─ Expected activity
  └─ Filter out

✓ Service name should NOT end with $
  └─ $ = machine accounts
  └─ Normal services
  └─ Filter out

✓ Account name should NOT be machine@domain
  └─ Machine account requests
  └─ Expected activity
  └─ Filter out

✓ Failure code should be 0x0
  └─ 0x0 = success
  └─ Failed requests are noise
  └─ Focus on successful requests
  └─ Failures mean account locked, permissions issues

✓ Encryption type must be 0x17
  └─ 0x17 = AES-256 (modern)
  └─ 0x18 = AES-128
  └─ 0x23 = RC4 (legacy)
  └─ Filter for modern encryption
```

---

### Detection Rules

**Alert on:**

```
✓ Kerberoasting
  └─ 4769 with encryption 0x17
  └─ User requesting service ticket
  └─ Unusual services
  └─ Multiple requests from same user

✓ Lateral movement
  └─ Service tickets for unusual services
  └─ Requests from unusual accounts
  └─ Cross-domain TGS requests

✓ Privilege escalation
  └─ Service tickets for sensitive services
  └─ From low-privilege accounts
  └─ To high-privilege systems

✓ DCSync activity
  └─ LDAP service tickets
  └─ Replication service tickets
  └─ From non-DC machines
```

---

### Implementation

```powershell
# Enable 4769 logging
Get-EventLog -LogName Security -InstanceId 4769 | 
  Where-Object {
    $_.Properties[1].Value -notlike '*krbtgt*' -and
    $_.Properties[1].Value -notlike '*$' -and
    $_.Properties[2].Value -notlike '*$' -and
    $_.Properties[3].Value -eq '0x0' -and
    $_.Properties[8].Value -eq '0x17'
  }
```

---

## Strategy 10: Service Account Security

**Service accounts = High-value targets**

### Problems with Standard Service Accounts

```
✗ Static passwords
  └─ Password rarely changed
  └─ Difficult to rotate
  └─ If compromised, persistent access

✗ Weak passwords
  └─ Often configured by developers
  └─ Simple, guessable passwords
  └─ Documented in config files
  └─ Vulnerable to offline cracking

✗ Credential storage
  └─ Stored in config files
  └─ In comments
  └─ In scripts
  └─ In version control
  └─ Easy to find
```

---

### Solution: Group Managed Service Accounts (gMSA)

**gMSA = Automatically managed service account credentials**

### gMSA Benefits

```
✓ Automatic password change
  └─ Password changed every 30 days
  └─ Automatically by domain
  └─ No manual rotation needed
  └─ No downtime

✓ Strong passwords
  └─ 120+ character random passwords
  └─ Changed frequently
  └─ Cannot be guessed
  └─ Difficult to crack offline

✓ No credential storage
  └─ Password in AD only
  └─ Retrieved by machine automatically
  └─ Not stored in config
  └─ Not in scripts/files

✓ Delegated SPN management
  └─ No manual SPN registration
  └─ Automatic SPN creation
  └─ Reduces admin errors
```

---

### gMSA Implementation

**Create gMSA:**

```powershell
# Create Key Distribution Service (KDS) root key
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))

# Create gMSA
New-ADServiceAccount -Name svc_app1 `
  -DNSHostName svc_app1.domain.local `
  -ServicePrincipalNames "HTTP/svc_app1.domain.local"

# Install gMSA on service machine
Install-ADServiceAccount -Identity svc_app1

# Configure service to use gMSA
# (In service configuration or app config)
```

---

### Kerberoasting Mitigation

**Summarizing all strategies:**

```
Mitigation:

1. Strong service account passwords
   └─ Use gMSA (automatic)
   └─ >35 characters
   └─ Regular rotation

2. Monitor 4769 events
   └─ Detect kerberoasting attempts
   └─ Alert on RC4 ticket requests
   └─ Alert on high volume from single user

3. Use AES encryption
   └─ Disallow RC4
   └─ Enforce AES-256
   └─ Makes offline cracking harder

4. Protect users from Protected Users group
   └─ Prevents Kerberoasting
   └─ No RC4 encryption available

5. Monitor service account usage
   └─ Alert on unusual access patterns
   └─ Track lateral movement
   └─ Monitor sensitive resource access
```

---

## Complete Defense Checklist

```
☐ Reduce Domain Admin count
☐ Restrict DA logon locations (only DC)
☐ Don't run services as DA
☐ Set AccountNotDelegated on DAs
☐ Add sensitive users to Protected Users group
☐ Implement PAW for admin tasks
☐ Deploy JIT administration
☐ Configure JEA for delegated admin
☐ Consider ESAE/Red Forest
☐ Monitor 4769 with proper filtering
☐ Deploy decoy objects
☐ Enable 4662 logging on honeypots
☐ Use Group Managed Service Accounts
☐ Enforce strong encryption (AES)
☐ Implement Zero Trust architecture
```

---

## References

- [Protected Users Group (Microsoft)](https://docs.microsoft.com/en-us/windows-server/security/credentials-protection-and-management/protected-users-security-group)
- [PAW Implementation Guide](https://docs.microsoft.com/en-us/security/compass/privileged-access-deployment)
- [JEA Documentation](https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/jea/overview)
- [ESAE to Privileged Access Strategy](https://docs.microsoft.com/en-us/security/compass/privileged-access-strategy)
- [Kerberoasting Detection (Microsoft)](https://docs.microsoft.com/en-us/defender-for-identity/lateral-movement-alerts)
- [Deploy-Deception](https://github.com/samratashok/Deploy-Deception)
- [Group Managed Service Accounts](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview)

---

*Next: Incident Response & Forensics*
