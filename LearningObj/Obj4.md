# Learning Objective 4 — Forest & Trust Enumeration

**Date:** 25 July 2026

---

## Task 1: Enumerate All Domains in moneycorp.local Forest

```powershell
Get-ForestDomain -Verbose
```

Lists all domains in the moneycorp.local forest.

---

## Task 2: Map Trusts of dollarcorp.moneycorp.local Domain

```powershell
Get-DomainTrust
```

Enumerates all trusts for the current domain (dollarcorp). Shows:
- Trust direction (inbound, outbound, bidirectional)
- Trust type (parent-child, external, forest)
- Transitive or non-transitive

---

## Task 3: Map External Trusts in moneycorp.local Forest

```powershell
Get-DomainTrust
```

Identifies external trusts (trusts to domains outside the forest). Filter for external trusts:

```powershell
Get-DomainTrust | Where-Object {$_.TrustType -eq "External"}
```

---

## Task 4: Identify External Trusts of dollarcorp & Enumerate Trusting Forest

**Question:** Can you enumerate trusts for a trusting forest (eurocorp.local)?

### Step 1: Enumerate Users in Trusting Forest

```powershell
Get-DomainUser -Domain eurocorp.local
```

Lists users in the eurocorp.local domain (trusting forest).

### Step 2: Map Trusts for All Domains in Trusting Forest

```powershell
Get-ForestDomain -Forest eurocorp.local | %{Get-DomainTrust -Domain $_.Name}
```

**Breakdown:**
- `Get-ForestDomain -Forest eurocorp.local` — lists all domains in eurocorp forest
- `%{Get-DomainTrust -Domain $_.Name}` — enumerates trusts for each domain

**Output shows:**
- All trusts established by eurocorp forest domains
- Trust directions and types
- Transitive/non-transitive relationships

---

## Key Concept

External trusts are bidirectional but non-transitive. If dollarcorp trusts eurocorp, you can:
- Enumerate eurocorp users and computers
- Map attack paths across the trust boundary
- Identify users with cross-forest privileges

---

## References

- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)

---

*Next: Learning Objective 5 — Credential Extraction & Persistence*
