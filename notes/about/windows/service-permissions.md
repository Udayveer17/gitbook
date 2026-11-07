---
icon: unlock
---

# Service Permissions

* Service permissions are critical part of Windows operating systems, and weak permissions may serve as potential threat vectors leading to execution of malicious executable or DLLs, privilege escalation etc.
* Most Services in Windows run with the LocalSystem privileges which is the highest level of access on an individual Windows OS.

### **Examining Services using sc**

The `sc qc` The command is used to query the service. If we wanted to query a service on a device over the network, we could specify the hostname or IP address immediately after `sc`.

```powershell
sc qc <service name> #queries service on the local machine

sc //hostname or ip of box query <ServiceName> # queries  service on a remote machine
```

We can also use sc to start and stop services.

```powershell
sc stop <service name>
```

### **Examining S**ervice permissions using `sc`

```powershell
sc sdshow wuauserv
```

At an initial glance, the output looks crazy. will look like a random string; the output is displayed in **Security Descriptor Definition Language (SDDL)**.

* Every named object in Windows is securable object
* Securable objects are those that have a security descriptor associated with them
* Security Descriptors identify the owner's primary group containing Discretionary Access Control List (DACL) and System Access Control List (SACL)
* DACL is used for controlling access to an object, and a SACL is used to account for and log access attempts.

`D: (A;;CCLCSWRPLORC;;;AU)`

1. D: - the proceeding characters are DACL permissions
2. AU: - defines the security principal Authenticated Users
3. A;; - access is allowed
4. CC - SERVICE\_QUERY\_CONFIG is the full name, and it is a query to the service control manager (SCM) for the service configuration
5. LC - SERVICE\_QUERY\_STATUS is the full name, and it is a query to the service control manager (SCM) for the current status of the service
6. SW - SERVICE\_ENUMERATE\_DEPENDENTS is the full name, and it will enumerate a list of dependent services
7. RP - SERVICE\_START is the full name, and it will start the service
8. LO - SERVICE\_INTERROGATE is the full name, and it will query the service for its current status
9. RC - READ\_CONTROL is the full name, and it will query the security descriptor of the service

As we read the security descriptor, it can be easy to get lost in the seemingly random order of characters, but recall that we are essentially viewing access control entries in an access control list.

### **Examine service permissions using PowerShell**

Using the `Get-Acl` PowerShell cmdlet, we can examine service permissions by targeting the path of a specific service in the registry.

```powershell
Get-ACL <path to the service in the registry> | Format-List
```
