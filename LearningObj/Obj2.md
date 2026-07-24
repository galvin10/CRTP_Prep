# Learning Objective 2 — ACL Enumeration

**Date:** 25 July 2026

---

## Task 1: Enumerate ACLs for Domain Admins Group

```powershell
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs -Verbose
```

Shows all permissions on Domain Admins group. Look for dangerous rights like GenericAll, WriteDACL, or ResetPassword.

---

## Task 2: Find Interesting ACLs Where studentx Has Permissions

```powershell
Find-InterestingDomainAcl -ResolveGUIDs | ?{$_.IdentityReferenceName -match "RDPUsers"}
```

Identify objects where your user/group has dangerous permissions.

**Filter options:**
```powershell
# By specific user
Find-InterestingDomainAcl -ResolveGUIDs | Where-Object {$_.IdentityReferenceName -match "studentx"}

# By permission type
Find-InterestingDomainAcl -ResolveGUIDs | Where-Object {$_.ActiveDirectoryRightName -match "GenericAll|WriteDACL"}

# Export to CSV
Find-InterestingDomainAcl -ResolveGUIDs | Export-Csv -Path "C:\AD\Tools\acls.csv" -NoTypeInformation
```

---

## Task 3: Use BloodHound to Visualize ACL Paths

1. Upload SharpHound data (ensure ACL collection methods included)
2. Search for user in BloodHound
3. Check "Outbound Object Control" — objects this user can control
4. Use Cypher query to find paths to Domain Admin:

```
MATCH (u:User {name:"STUDENTX@DOLLARCORP.LOCAL"}) 
MATCH (da:Group {name:"DOMAIN ADMINS@DOLLARCORP.LOCAL"}) 
MATCH p=shortestPath((u)-[r:GenericAll|WriteDACL|AddMember]->(da)) 
RETURN p
```

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [BloodHound CE](https://github.com/SpecterOps/BloodHound)

---

*Next: Learning Objective 3 — Kerberoasting & Credential Extraction*
