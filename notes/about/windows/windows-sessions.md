---
icon: circle-user-clock
---

# Windows Sessions

#### Interactive

* An interactive session is initiated by a user authenticating to a local or domain system by entering their credentials.
* It can be initiated by logging in directly to the system, by requesting a secondary logon session using runas, or through an RDP connection

#### Non-Interactive

* Non-interactive sessions are initiated by non-interactive accounts, they don't require credentials
* There are 3 types of non-interactive accounts :
  * LocalSystem Account
  * Local Service Account
  * Network Service Account
* These accounts are used by the OS to start/stop services during boot or to run scheduled tasks.

| Account                                                                  | Description                                         |
| ------------------------------------------------------------------------ | --------------------------------------------------- |
| Local System Account                                                     | Also known as the `NT AUTHORITY\\SYSTEM`            |
| account, this is the most powerful account in Windows systems. It is     |                                                     |
| used for a variety of OS-related tasks, such as starting Windows         |                                                     |
| services. This account is more powerful than accounts in the local       |                                                     |
| administrators group.                                                    |                                                     |
| Local Service Account                                                    | Known as the `NT AUTHORITY\\LocalService`           |
| account, this is a less privileged version of the SYSTEM account and     |                                                     |
| has similar privileges to a local user account. It is granted limited    |                                                     |
| functionality and can start some services.                               |                                                     |
| Network Service Account                                                  | This is known as the `NT AUTHORITY\\NetworkService` |
| account and is similar to a standard domain user account. It has         |                                                     |
| similar privileges to the Local Service Account on the local machine. It |                                                     |
| can establish authenticated sessions for certain network services.       |                                                     |
