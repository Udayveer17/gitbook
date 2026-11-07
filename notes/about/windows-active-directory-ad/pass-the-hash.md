---
icon: wind
---

# Pass The Hash

During internal security tests, auditors often need to move from one computer to another (called **lateral movement**) to find useful data and gain higher access. A common trick used in these tests is **Pass-the-Hash** it lets someone reuse a stored password hash to log into other machines and try to become an administrator.

> To understand pass-the-hash you mast under stand how NTLM protocol work's&#x20;
>
> [https://www.udayxd.xyz/notes/network-protocols/ntlm-protocol](https://www.udayxd.xyz/notes/network-protocols/ntlm-protocol)

### What is Pass the Hash?

**Pass the Hash (PTH)** is an authentication attack where an attacker uses a stolen password hash (specifically the NT hash in Windows) to authenticate to systems _without needing to crack the password_. Windows allows authentication using just the hash itself.

### Why This Works

In Windows authentication, the system stores password hashes (NT hashes) rather than plain passwords. When you authenticate, Windows can verify you by checking your hash. The vulnerability: **if you have someone's NT hash, you can authenticate as them without knowing their actual password**.

### Two Attack Scenarios Explained

#### Scenario 1: Identical Local Administrator Accounts

**The Setup:**

* IT departments often create a "master image" of Windows for efficiency
* This master has a configured local Administrator account
* Every workstation deployed from this master has the **same Administrator account with the same password**
* Same password = same NT hash across all machines

**The Attack:**

1. Attacker compromises one workstation
2. Extracts the local Administrator's NT hash (e.g., `20cc650a5ac276a1cfc22fbc23beada1`)
3. Uses this hash to authenticate to ANY other workstation built from the same master
4. Result: Administrative access to potentially dozens or hundreds of machines

**Example given:**&#x20;

## Using Impacket's psexec.py with the stolen hash

psexec.py -hashes :20cc650a5ac276a1cfc22fbc23beada1 Administrator@target-machine

#### Scenario 2: Domain Group with Local Admin Rights

**The Setup:**

* IT creates a domain group (e.g., "HelpDesk")
* This group is added to the local Administrators group on all workstations via Group Policy (GPO)
* Anyone in HelpDesk can administer any workstation

**The Attack:**

1. Attacker compromises one user account in the HelpDesk group (e.g., user "jsnow")
2. Extracts jsnow's NT hash (e.g., `89db9cd74150fc8d8559c3c19768ca3f`)
3. Uses this domain user's hash to authenticate to ANY machine where HelpDesk has admin rights
4. The domain controller validates the authentication
5. The target machine sees the user is in HelpDesk group → grants admin access

**Why this is more powerful:**

* Works across ALL workstations regardless of which master image was used
* Domain groups are typically added via GPO, so coverage is broader
* One compromised domain account = potential access to entire fleet

### The Authentication Flow (Scenario 2)

Attacker → \[Uses jsnow's hash] → Target Workstation ↓ Asks Domain Controller: "Is this hash valid?" ↓ DC responds: "Yes, that's jsnow, member of HelpDesk" ↓ Workstation: "HelpDesk is in local Admins, grant access"

### Why This is Dangerous

1. **No password cracking needed** - hashes work directly
2. **Lateral movement** - one compromise leads to many
3. **Persistence** - unless passwords change, hashes remain valid
4. **Difficult to detect** - looks like legitimate authentication.

### Pass the Hash Limitations

While Pass the Hash works when NTLM authentication is enabled (which is the default), Windows has built-in protections that can restrict what an attacker can actually do after authenticating.

### Access Tokens and UAC (User Account Control)

**How Windows manages privileges:**

Windows uses **Access Tokens** to control permissions - they define "who can do what."

**For members of the Administrators group:**

* They receive **two tokens**:
  1. A **standard user token** (limited privileges) - used by default
  2. An **administrator token** (full privileges) - used only when needed

**Locally on a machine:** When an administrator tries to perform administrative tasks, Windows shows the UAC prompt:

```
[UAC Window appears]
"Do you want to allow this app to make changes to your device?"
```

This elevates from the standard token to the admin token.

> will cover UAC in up-coming notes&#x20;

