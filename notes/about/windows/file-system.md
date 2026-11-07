---
icon: folder
---

# File System

There are 5 types of Windows file systems:

* FAT32
* FAT16
* FAT12
* exFAT
* NTFS

#### FAT32

* FAT32 (File Allocation Table) is widely used across many types of storage devices such as USB memory sticks and SD cards but can also be used to format hard drives.
*   The "32" in the name refers to the fact that FAT32 uses 32 bits of data for identifying data clusters on a storage device.

    **`Pros of FAT32:`**

    * Device compatibility - it can be used on computers, digital cameras, gaming consoles, smartphones, tablets, and more.
    * Operating system cross-compatibility - It works on all Windows operating systems starting from Windows 95 and is also supported by MacOS and Linux.

    **`Cons of FAT32:`**

    * Can only be used with files that are less than 4GB.
    * No built-in data protection or file compression features.
    * Must use third-party tools for file encryption.

#### NTFS

* NTFS (New Technology File System) is the default Windows file system since Windows NT 3.1. In addition to making up for the shortcomings of FAT32
*   NTFS also has better support for metadata and better performance due to improved data structuring.

    **`Pros of NTFS:`**

    * NTFS is reliable and can restore the consistency of the file system in the event of a system failure or power loss.
    * Provides security by allowing us to set granular permissions on both files and folders.
    * Supports very large-sized partitions.
    * Has journaling built-in, meaning that file modifications (addition, modification, deletion) are logged.

    **`Cons of NTFS:`**

    * Most mobile devices do not support NTFS natively.
    * Older media devices such as TVs and digital cameras do not offer support for NTFS storage devices.

### Permissions

The NTFS file system has many basic and advanced permissions. Some of the key permission types are:

| Permission Type                                                          | Description                                                                                                                |
| ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| Full Control                                                             | Allows reading, writing, changing, deleting of files/folders.                                                              |
| Modify                                                                   | Allows reading, writing, and deleting of files/folders.                                                                    |
| List Folder Contents                                                     | Allows for viewing and listing folders and subfolders as well as executing files. Folders only inherit this permission.    |
| Read and Execute                                                         | Allows for viewing and listing files and subfolders as well as executing files. Files and folders inherit this permission. |
| Write                                                                    | Allows for adding files to folders and subfolders and writing to a file.                                                   |
| Read                                                                     | Allows for viewing and listing of folders and subfolders and viewing a file's contents.                                    |
| Traverse Folder                                                          | This                                                                                                                       |
| allows or denies the ability to move through folders to reach other      |                                                                                                                            |
| files or folders. For example, a user may not have permission to list    |                                                                                                                            |
| the directory contents or view files in the documents or web apps        |                                                                                                                            |
| directory in this example                                                |                                                                                                                            |
| c:\users\bsmith\documents\webapps\backups\backup\_02042020.zip but with  |                                                                                                                            |
| Traverse Folder permissions applied, they can access the backup archive. |                                                                                                                            |

Files and folders inherit the NTFS permissions of their parent folder for ease of administration, so administrators do not need to explicitly set permissions for each file and folder, as this would be extremely time-consuming.

### **Integrity Control Access Control List (icacls)**

**icacls** is a powerful Windows command-line utility used to display, modify, back up, or restore Discretionary Access Control Lists (DACLs) on files and folders

Key capabilities of `icacls` include:

* **Manage Permissions:** Grant (`/grant`), deny (`/deny`), and remove (`/remove`) specific user or group access rights to files and folders.
* **Modify Ownership:** Change the owner of a file or folder using the `/setowner` parameter.
* **Inheritance Control:** Enable, disable, or reset inheritance (`/inheritance:e|d|r`) to restore default permissions.
* **Recursive Operations:** Apply changes to all files within a folder and its subfolders using the `/t` flag.
* **Backup and Restore:** Save existing ACLs to a file for later restoration using `/save` and `/restore`.
* **Integrity Levels:** Set integrity levels for files to interact with Mandatory Integrity Control (MIC).

**Common Permission Masks:**

* **F** (Full Access)
* **M** (Modify Access)
* **RX** (Read and Execute Access)
* **R** (Read-only Access)
* **W** (Write-only Access)

#### **The resource access level**

`(CI)`: container inherit

* **Purpose:** Ensures that new subfolders inherit the permissions of the parent folder.
* **Applicability:** Applies only to containers (directories/folders).

**Example Command**

```jsx
icacls "C:\\htb" /grant uday:(CI)R #This will give me read perms on all sub folders of htb
```

`(OI)`: object inherit

* **Purpose: The** purpose is to ensure that Access Control Entries (ACEs), specifically permissions applied to a parent directory, are inherited by all **files** (objects) created within that folder.

> Note: (OI) does not affect existing files it applies to new files created within the directory.

**Example Command**

```jsx
icacls "C:\\htb" /grant uday:(OI)R #User can read all files, but has no access to subfolders
```

`(IO)`: inherit only

* **Purpose: Applies to Children, Not Parent:** When you apply a permission with `(IO)`, The current directory's/folders security descriptor ignores this rule. However, any new files or subfolders created within it will inherit this permission.

**Example Command**

```powershell
icacls "C:\\htb" /grant uday:(IO)R # User gets read access only to child files and folders, Not on the parent folder 
```

`(NP)`: do not propagate inherit

* **Purpose:** It prevents deep inheritance, restricting permission scope to one level.
* Like it applies to children, but **grandchildren and deeper do not**

**Example Command**

```powershell
icacls "C:\\htb" /grant uday:(NP)R # User gets read access on immediate children, but not on nested folders.
```

`(I)`: permission inherited from parent container

* **(I)** signifies that an Access Control Entry (ACE) is **Inherited from the parent container**.
