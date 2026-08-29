CRTP Practice Documentation --- Enumeration to Objective 5

Date: 30 August 2026
Scope: CRTP practice completed through Objective 5, including
Jenkins.

1. Domain Enumeration

Get-Domain
Get-Forest
Get-DomainUser
Get-DomainGroup
Get-DomainComputer
Get-DomainOU

Privileged Groups

Get-DomainGroupMember -Identity "Domain Admins"
Get-DomainGroupMember -Identity "Enterprise Admins" -Domain moneycorp.local

Domain Admins → privileged group for one domain.

Enterprise Admins → forest-level privileged group.

Get-DomainGroupMember → shows who belongs to a group.

2. OU / Computer Enumeration

(Get-DomainOU -Identity DevOps).distinguishedname |
%{Get-DomainComputer -SearchBase $_} |
select name

Finds the DevOps OU and lists computers under it.

3. ACL Enumeration

Get-DomainObjectAcl -Identity "Domain Admins" -Verbose

Also tested:

Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs
Find-InterestingDomainAcl

-ResolveGUIDs makes AD permission GUIDs readable. In the lab, GUID
resolution produced a PowerView Get-DomainGUIDMap/enum error, while
the normal ACL query worked.

4. BloodHound / Neo4j

Reviewed BloodHound Legacy and Neo4j setup.

cd C:\AD\Tools\neo4j-community-4.4.5-windows\neo4j-community-4.4.5\bin
.\neo4j.bat install-service
.\neo4j.bat start

Note: PowerShell requires .\ for a program in the current
directory. Neo4j service installation requires administrator privileges.

BloodHound helps visualize AD relationships and interesting privilege
paths.

5. Jenkins

Practiced the Jenkins portion through Objective 5 as part of the
CRTP lab workflow.

Takeaway: Jenkins is an important application/server to identify
during enumeration and investigate according to the lab objective.

Key Takeaways

Enumerate the domain and forest first.

Identify privileged groups and members.

Enumerate computers and OUs.

Review interesting ACL permissions.

Use BloodHound to visualize AD relationships.

Investigate important application servers such as Jenkins.

Record errors and fixes during practice.

Progress: Completed practice through Objective 5, including
Jenkins.
