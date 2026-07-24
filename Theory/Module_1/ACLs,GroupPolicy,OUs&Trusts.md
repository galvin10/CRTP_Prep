#  Domain Enumeration — ACLs, Group Policy, OUs & Trusts

**Date:** 25 July 2026

---

## Access Control Model (ACL Basics)

**Components:**
- **Access Tokens** — security context of a process (identity and privileges of user)
- **Security Descriptors** — SID of owner, DACL (permissions), SACL (auditing)

**ACL Types:**
- **DACL** — defines permissions trustees have on an object
- **SACL** — logs success/failure audit messages when object is accessed

---

## Group Policy Overview

Group Policy manages configuration centrally in AD.

**Policy Settings:**
- **For Computers:** security settings, startup/shutdown scripts, assigned applications
- **For Users:** security settings, logon/logoff scripts, assigned applications

**GPO:** Virtual collection of policy settings linked to domains, sites, or OUs. Can be abused for privilege escalation, backdoors, persistence.

---

## Trust Types

**One-way Trust:** Users in trusted domain can access resources in trusting domain (not reverse).

**Two-way Trust:** Both domains can access resources in each other.

**Transitive:** Can extend to other domains. Default for intra-forest trusts (parent-child, tree-root).

**Non-transitive:** Cannot extend to other domains. Default for external trusts between different forests.

**Default Trusts:**
- Parent-child: Created automatically between new domain and predecessor (two-way transitive)
- Tree-root: Created between new domain tree and forest root (two-way transitive)

**External Trusts:** Between different forests, one-way or two-way, non-transitive.

**Forest Trusts:** Between forest roots, one-way or two-way transitive, cannot extend to third forest.

---

## Enumerate ACLs

```powershell
# ACL for specific user
Get-DomainObjectAcl -SamAccountName student1 -ResolveGUIDs

# ACL using LDAP path
Get-DomainObjectAcl -SearchBase "LDAP://CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local" -ResolveGUIDs -Verbose

# Using AD Module (no GUID resolution)
(Get-Acl 'AD:\CN=Administrator,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local').Access

# Find interesting ACEs
Find-InterestingDomainAcl -ResolveGUIDs

# ACLs on file share path
Get-PathAcl -Path "\\dcorp-dc.dollarcorp.moneycorp.local\sysvol"
```

---

## Enumerate Group Policy

```powershell
# List all GPOs
Get-DomainGPO
Get-DomainGPO -ComputerIdentity dcorp-student1

# Get GPOs using Restricted Groups or groups.xml
Get-DomainGPOLocalGroup

# Users in local group via GPO
Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity dcorp-student1

# Machines where user is member of group via GPO
Get-DomainGPOUserLocalGroupMapping -Identity student1 -Verbose
```

---

## Enumerate OUs

```powershell
# List all OUs
Get-DomainOU
Get-ADOrganizationalUnit -Filter * -Properties *

# Get GPO applied on OU (use GPOname from gplink attribute)
Get-DomainGPO -Identity "{0D1CC23D-1F20-4EEE-AF64-D99597AE2A6E}"
```

---

## Domain Trust Mapping

```powershell
# Domain trusts (current domain)
Get-DomainTrust
Get-DomainTrust -Domain us.dollarcorp.moneycorp.local
Get-ADTrust
Get-ADTrust -Identity us.dollarcorp.moneycorp.local
```

---

## Forest Mapping

```powershell
# Forest details
Get-Forest
Get-Forest -Forest eurocorp.local
Get-ADForest
Get-ADForest -Identity eurocorp.local

# All domains in forest
Get-ForestDomain
Get-ForestDomain -Forest eurocorp.local
(Get-ADForest).Domains

# Global catalogs
Get-ForestGlobalCatalog
Get-ForestGlobalCatalog -Forest eurocorp.local
Get-ADForest | Select -ExpandProperty GlobalCatalogs

# Forest trusts
Get-ForestTrust
Get-ForestTrust -Forest eurocorp.local
Get-ADTrust -Filter 'msDS-TrustForestTrustInfo -ne "$null"'
```

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [Active Directory PowerShell Module](https://learn.microsoft.com/en-us/powershell/module/activedirectory/)

---

*Next: Learning Objective 5 — Kerberoasting & Credential Extraction*
