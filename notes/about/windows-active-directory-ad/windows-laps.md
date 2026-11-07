---
icon: laptop
---

# Windows LAPS

## Windows LAPS (Local Administrator Password Solution)

### What is Windows LAPS?

Windows LAPS (Local Administrator Password Solution) is a security feature that automatically manages and rotates passwords for local administrator accounts on domain-joined devices. It generates unique, random passwords for each device's local admin account, stores them securely in Active Directory or Microsoft Entra ID, and rotates them on a regular schedule.

**Why LAPS matters:** Without LAPS, organizations often use the same local administrator password across all computers. If an attacker compromises one machine, they can use that password to move laterally across the entire network. LAPS eliminates this risk by ensuring each machine has a unique local admin password.

LAPS can also manage DSRM (Directory Services Restore Mode) passwords on domain controllers, adding another layer of security to critical infrastructure.

### How LAPS Works

Active Directory stores the local administrator passwords for all LAPS-enabled computers in the domain. The passwords are written to a specific attribute on each computer object in AD.

Administrators typically create a security group (such as "LAPS Password Readers") and grant members the appropriate permissions to read these passwords. This delegation allows trusted users to retrieve local admin credentials when needed for legitimate administrative tasks, while maintaining an audit trail of who accessed which passwords.

**Important security note:** Reading LAPS passwords is a sensitive operation that generates audit logs. Security teams monitor for excessive or unauthorized LAPS password retrievals as this could indicate malicious activity.

### Reading LAPS Passwords

To read a LAPS password, use the `Get-ADComputer` cmdlet and specifically request the `ms-mcs-admpwd` property:

```powershell
Get-ADComputer DC01 -Property 'ms-mcs-admpwd'
```

**Example output:**

```powershell
*Evil-WinRM* PS C:\Users\svc_deploy\Documents> Get-ADComputer DC01 -Property 'ms-mcs-admpwd'

DistinguishedName : CN=DC01,OU=Domain Controllers,DC=timelapse,DC=htb
DNSHostName       : dc01.timelapse.htb
Enabled           : True
ms-mcs-admpwd     : uM[3va(s870g6Y]9i]6tMu{j
Name              : DC01
ObjectClass       : computer
ObjectGUID        : 6e10b102-6936-41aa-bb98-bed624c9b98f
SamAccountName    : DC01$
SID               : S-1-5-21-671920749-559770252-3318990721-1000
UserPrincipalName :
```

This returns the computer object with the `ms-mcs-admpwd` field containing the current local administrator password. In this example, the local administrator password for DC01 is `uM[3va(s870g6Y]9i]6tMu{j`.

### Understanding the `ms-mcs-admpwd` Attribute

`ms-mcs-admpwd` is the Active Directory attribute where LAPS stores the local administrator password. When LAPS is enabled on a computer, the machine automatically updates this attribute in its AD computer object with its current local admin password.

**Breaking down the name:**

* **ms** = Microsoft
* **mcs** = Microsoft Corporate Solutions (the team that originally developed LAPS)
* **admpwd** = Administrator Password

By default, this attribute is protected and only readable by Domain Admins and explicitly delegated users or groups. When you query a computer object in AD, this property isn't returned by default—you must specifically request it using the `-Property` parameter.

### Required Permissions

To read LAPS passwords, a user needs one of the following:

* Domain Admin privileges (has access by default)
* "All extended rights" permission on computer objects
* The specific "Read ms-mcs-admpwd" permission delegated on computer objects or OUs

These permissions should be carefully controlled and regularly audited to prevent unauthorized access to local administrator credentials.
