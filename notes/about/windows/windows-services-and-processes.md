---
icon: message-question
---

# Windows Services & Processes

### **Windows Services**

* Services are Processes/Applications which runs in the background without the user's knowledge.

Windows services are managed via the Service Control Manager (SCM) system, accessible via the `services.msc` MMC add-in.

#### **Querying services by cmd line**

```powershell
Get-Service | ? {$_.Status -eq "Running"} 
```

#### Categories of Windows Services

Windows has three categories of services :

* Local Services
* Network Services
* System Services

In Windows, we have some [critical system services](https://docs.microsoft.com/en-us/windows/win32/rstmgr/critical-system-services) that cannot be stopped and restarted without a system restart.

| Service                                                                 | Description                                                                    |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| smss.exe                                                                | Session Manager SubSystem. Responsible for handling sessions on the system.    |
| csrss.exe                                                               | Client Server Runtime Process. The user-mode portion of the Windows subsystem. |
| wininit.exe                                                             | Starts                                                                         |
| the Wininit file .ini file that lists all of the changes to be made to  |                                                                                |
| Windows when the computer is restarted after installing a program.      |                                                                                |
| logonui.exe                                                             | Used for facilitating user login into a PC                                     |
| lsass.exe                                                               | The                                                                            |
| Local Security Authentication Server verifies the validity of user      |                                                                                |
| logons to a PC or server. It generates the process responsible for      |                                                                                |
| authenticating users for the Winlogon service.                          |                                                                                |
| services.exe                                                            | Manages the operation of starting and stopping services.                       |
| winlogon.exe                                                            | Responsible                                                                    |
| for handling the secure attention sequence, loading a user profile on   |                                                                                |
| logon, and locking the computer when a screensaver is running.          |                                                                                |
| System                                                                  | A background system process that runs the Windows kernel.                      |
| svchost.exe with RPCSS                                                  | Manages                                                                        |
| system services that run from dynamic-link libraries (files with the    |                                                                                |
| extension .dll) such as "Automatic Updates," "Windows Firewall," and    |                                                                                |
| "Plug and Play." Uses the Remote Procedure Call (RPC) Service (RPCSS).  |                                                                                |
| svchost.exe with Dcom/PnP                                               | Manages                                                                        |
| system services that run from dynamic-link libraries (files with the    |                                                                                |
| extension .dll) such as "Automatic Updates," "Windows Firewall," and    |                                                                                |
| "Plug and Play." Uses the Distributed Component Object Model (DCOM) and |                                                                                |
| Plug and Play (PnP) services.                                           |                                                                                |

#### **Processes**

> Processes run in the background on Windows systems. They either run automatically as part of the Windows operating system or are started by other installed applications.

#### **Local Security Authority Subsystem Service (LSASS)**

* `lsass.exe` is the process that is responsible for enforcing the security policy on Windows systems.
* When a user attempts to log on to the system, this process verifies their log on attempt and creates access tokens based on the user's permission levels.
* LSASS is also responsible for user account password changes.
* All events associated with this process (logon/logoff attempts, etc.) are logged within the Windows Security Log.
* Passwords/Hashes can be extracted from the `lsass` process
