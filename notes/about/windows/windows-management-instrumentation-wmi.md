---
icon: users-gear
---

# Windows Management Instrumentation (WMI)

* WMI provides an interface to query and control system components in Windows, such as processes, registry, settings, event logs, hardware, and much more.
* It allows for device and application management.

#### How WMI Works?

* WMI is based on the CIM (Common Information Model ) that treats system components(Here system components refers to processes, settings etc) as objects with properties and methods, much like a class in OOP.
* WMI interacts with these objects ( which are system components ) via the WMI provider model.
  * Providers are DLLs or COM objects that serve as an intermediary between WMI and the managed resources. ( eg. Win32\_LogicalDIsk, Win32\_Process etc )
* WMI has a local database of CMI schema and instance data
* WMI also supports system management across remote devices via DCOM

#### Use Cases of WMI

1. System Monitoring: CPU, RAM, disk info
2. Remote/Local system management
3. Automation of Administrative Tasks
4. Log/event query
5. setting/modifying system properties
6. code execution
7. Scheduling processes

#### WMI Components

| **Component Name**                                                      | **Description**                                                                                   |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| WMI service                                                             | The                                                                                               |
| Windows Management Instrumentation process, which runs automatically at |                                                                                                   |
| boot and acts as an intermediary between WMI providers, the WMI         |                                                                                                   |
| repository, and managing applications.                                  |                                                                                                   |
| Managed objects                                                         | Any logical or physical components that can be managed by WMI.                                    |
| WMI providers                                                           | Objects that monitor events/data related to a specific object.                                    |
| Classes                                                                 | These are used by the WMI providers to pass data to the WMI service.                              |
| Methods                                                                 | These                                                                                             |
| are attached to classes and allow actions to be performed. For example, |                                                                                                   |
| methods can be used to start/stop processes on remote machines.         |                                                                                                   |
| WMI repository                                                          | A database that stores all static data related to WMI.                                            |
| CIM Object Manager                                                      | The system that requests data from WMI providers and returns it to the application requesting it. |
| WMI API                                                                 | Enables applications to access the WMI infrastructure.                                            |
| WMI Consumer                                                            | Sends queries to objects via the CIM Object Manager.                                              |

#### WMI Abuse

WMI allows users to trigger actions when certain events occur (eg; when a new process starts). Attackers can leverage this as a potential persistence mechanism
