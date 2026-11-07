---
icon: angle
---

# NTFS vs. Share Permissions

The Server Message Block protocol (SMB) is used in Windows to connect shared resources like files and printers. It is used in large, medium, and small enterprise environments.

NTFS permissions and share permissions are often understood to be the same. Please know that they are not the same but often apply to the same shared resource.

#### Share permissions

| Permission                                                              | Description                                                            |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `Full Control`                                                          | Users                                                                  |
| are permitted to perform all actions given by Change and Read           |                                                                        |
| permissions as well as change permissions for NTFS files and subfolders |                                                                        |
| `Change`                                                                | Users are permitted to read, edit, delete and add files and subfolders |
| `Read`                                                                  | Users are allowed to view file & subfolder contents                    |

#### NTFS special permissions

| Permission                                                               | Description                                                                                                                 |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `Full control`                                                           | Users                                                                                                                       |
| are permitted or denied permissions to add, edit, move, delete files     |                                                                                                                             |
| & folders as well as change NTFS permissions that apply to all           |                                                                                                                             |
| permitted folders                                                        |                                                                                                                             |
| `Traverse folder / execute file`                                         | Users                                                                                                                       |
| are permitted or denied permissions to access a subfolder within a       |                                                                                                                             |
| directory structure even if the user is denied access to contents at the |                                                                                                                             |
| parent folder level. Users may also be permitted or denied permissions   |                                                                                                                             |
| to execute programs                                                      |                                                                                                                             |
| `List folder/read data`                                                  | Users                                                                                                                       |
| are permitted or denied permissions to view files and folders contained  |                                                                                                                             |
| in the parent folder. Users can also be permitted to open and view       |                                                                                                                             |
| files                                                                    |                                                                                                                             |
| `Read attributes`                                                        | Users                                                                                                                       |
| are permitted or denied permissions to view basic attributes of a file   |                                                                                                                             |
| or folder. Examples of basic attributes: system, archive, read-only, and |                                                                                                                             |
| hidden                                                                   |                                                                                                                             |
| `Read extended attributes`                                               | Users                                                                                                                       |
| are permitted or denied permissions to view extended attributes of a     |                                                                                                                             |
| file or folder. Attributes differ depending on the program               |                                                                                                                             |
| `Create files/write data`                                                | Users are permitted or denied permissions to create files within a folder and make changes to a file                        |
| `Create folders/append data`                                             | Users                                                                                                                       |
| are permitted or denied permissions to create subfolders within a        |                                                                                                                             |
| folder. Data can be added to files but pre-existing content cannot be    |                                                                                                                             |
| overwritten                                                              |                                                                                                                             |
| `Write attributes`                                                       | Users are permitted or denied to change file attributes. This permission does not grant access to creating files or folders |
| `Write extended attributes`                                              | Users                                                                                                                       |
| are permitted or denied permissions to change extended attributes on a   |                                                                                                                             |
| file or folder. Attributes differ depending on the program               |                                                                                                                             |
| `Delete subfolders and files`                                            | Users are permitted or denied permissions to delete subfolders and files. Parent folders will not be deleted                |
| `Delete`                                                                 | Users are permitted or denied permissions to delete parent folders, subfolders and files.                                   |
| `Read permissions`                                                       | Users are permitted or denied permissions to read permissions of a folder                                                   |
| `Change permissions`                                                     | Users are permitted or denied permissions to change permissions of a file or folder                                         |
| `Take ownership`                                                         | Users                                                                                                                       |
| are permitted or denied permission to take ownership of a file or        |                                                                                                                             |
| folder. The owner of a file has full permissions to change any           |                                                                                                                             |
| permissions                                                              |                                                                                                                             |

#### Application of NTFS permissions and share permissions

* NTFS permissions apply to the system where the folder and files are kept
* Share permissions apply when the folder is being accessed through the SMB, typically from a different system over the network along with NTFS permissions

This means someone accessing the system via RDP or physically will encounter NTFS permissions and a person accessing the folder via SMB will encounter share permissions and NTFS permissions

#### **Creating a Network Share**

* Similar to NTFS permissions, there is an access control list (ACL) for shared resources. We can consider this the SMB permissions list.
* Keep in mind that with shared resources, both the SMB and NTFS permissions lists apply to every resource that gets shared in Windows.
* The ACL contains `access control entries` (`ACEs`). Typically these ACEs are made up of `users` & `groups` (also called security principals) as they are a suitable mechanism for managing and tracking access to shared resources.

#### **Windows Defender Firewall Considerations**

* When a Windows System is a part of a workgroup, all NetLogon requests are authenticated against that particular windows system's SAM database.
* When a Windows System is a part of an AD environment, all NetLogon requests are authenticated against AD.
