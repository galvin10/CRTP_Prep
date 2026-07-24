#  Learning Objective 3 — OU & GPO Enumeration

**Date:** 22 July 2026
---

## Task 1: List All OUs

```powershell
Get-DomainOU | Select name
```

Lists all Organizational Units in the domain.

---

## Task 2: List Computers in DevOps OU

```powershell
Get-DomainOU -Identity DevOps
```

Get OU details. Then find computers:

```powershell
(Get-DomainOU -Identity DevOps).distinguishedname | %{Get-DomainComputer -SearchBase $_} | Select name
```

---

## Task 3: List All GPOs

```powershell
Get-DomainGPO | Select displayname
```

Lists all Group Policy Objects in the domain.

---

## Task 4: Enumerate GPOs Applied to DevOps OU

```powershell
Get-DomainOU -Identity DevOps
```

Get OU details and note the GPO links. Then query specific GPO:

```powershell
Get-DomainGPO -Identity '<GpoLinkValue>'
```

Replace `<GpoLinkValue>` with the actual GPO link from the OU output.

---

## Task 5: Enumerate ACLs for AppLocker and DevOps GPO

Use BloodHound:

1. Search for "DevOps" GPO in BloodHound
2. Check "Inbound Object Control" — who can modify this GPO
3. Search for "AppLocker" GPO
4. Analyze permissions and exploitation paths

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [BloodHound CE](https://github.com/SpecterOps/BloodHound)

---

*Next: Learning Objective 4 — Kerberoasting & Credential Extraction*
