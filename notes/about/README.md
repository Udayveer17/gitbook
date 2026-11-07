---
icon: windows
---

# Introduction to Active Directory

## Why Active Directory(AD)?

Active Directory (AD) is a directory service for Windows network environments. it have an a distributed structure that allows for centralized management of an organization's resources including users, computers, groups, network devices, file shares, group policies, devices and trusts

> AD is essentially a sizeable read-only database accessible to all users within the domain, regardless of their privilege level A basic AD user account with no privileges can enumerate most objects within AD

This thing makes it extremely important to properly secure an AD Set-up because if ANY user can be used to enumerate in the domain and hunt for misconfiguration and flaws thoroughly regardless of their privilege level, it can be a big problem. Also, multiple attacks can be performed with only a standard domain user , showing the importance of a defense strategy .

Active Directory makes information easy to find and use for administrators and users. AD is highly scalable, supports millions of objects per domain, and allows the creation of additional domains as an organization grows

<figure><img src=".gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>

For this reason, we must understand how Active Directory is set up and the basics of administration before attempting to attack it. It's always easier to "break" things if we already know how to build them.

***

## **Active Directory Structure**

so as we know that Active Directory is arranged in a tee structure, with in the forest at the top contain one or more domains, which themselves have nested subdomains (”**nested subdomains**” refers to the hierarchical arrangement of child domains branching out from a parent domain, forming a domain tree). A forest is the security boundary within which all objects are under administrative control and a forest may have multiple domains in it and a domain maybe have further child or sub-domains. A domain is logical grouping of network objects like users, computers, and groups, managed under a common set of policies and security settings and it have many in-build thing such as Organizational Units (OUs) `Domain Controllers`, `Users`, `Computers`, and new OUs can be created as required. OUs may contain objects and sub-OUs, allowing for the assignment of different group policies.

> It is common to see multiple domains (or forests) linked together via trust relationships in organizations that perform a lot of acquisitions. It is often quicker and easier to create a trust relationship with another domain/forest than recreate all new users in the current domain.

`this lines below are taken from htb module they explain good`

The graphic below shows two forests, `INLANEFREIGHT.LOCAL` and `FREIGHTLOGISTICS.LOCAL`. The two-way arrow represents a bidirectional trust between the two forests, meaning that users in `INLANEFREIGHT.LOCAL` can access resources in `FREIGHTLOGISTICS.LOCAL` and vice versa. We can also see multiple child domains under each root domain. In this example, we can see that the root domain trusts each of the child domains, but the child domains in forest A do not necessarily have trusts established with the child domains in forest B. This means that a user that is part of `admin.dev.freightlogistics.local` would NOT be able to authenticate to machines in the `wh.corp.inlanefreight.local` domain by default even though a bidirectional trust exists between the top-level `inlanefreight.local` and `freightlogistics.local` domains. To allow direct communication from `admin.dev.freightlogistics.local` and `wh.corp.inlanefreight.local`, another trust would need to be set up.

<figure><img src=".gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

***

## **Terminology**

Before we go any further, let's take a step back and define some key terminology that will be used throughout my notes and in general when dealing with Active Directory in any capacity.

#### Object&#x20;

an **object** is a fundamental unit representing a single network resource, such as a user, computer, printer, or group. Each object is defined by a set of **attributes**, which are pieces of information about it, like a user's name, email address, or a computer's operating system.&#x20;

in short ANY resource present within an Active Directory environment such as OUs, printers, users, domain controllers, etc. are object.

#### Attributes

In Active Directory, attributes are fields that store specific information about an object, such as a user's email address or a computer's hostname and DNS name. All attributes in AD have an associated LDAP name that can be used when performing LDAP queries, such as `displayName` for `Full Name` and `given name` for `First Name`.

#### Schema

Schema is the blueprint that defines the structure and rules for all objects and attributes within an AD forest. It specifies what types of objects (like users, computers, and groups) can exist and what properties (**attributes**) they can have. The schema ensures consistency and controls how information is stored and retrieved across all domain controllers. It contains formal definitions for every object class (like `user` or `group`) and attribute (like `givenName` or `cn`) that can exist in AD

#### Domain

A domain is a logical group of objects such as computers, users, OUs, groups, etc. We can think of each domain as a different city within a state or country. Domains can operate entirely independently of one another or be connected via trust relationships. `domains can operate independently`, **meaning** they each manage their own users, resources, and authentication. Alternatively, they can be linked through trust relationships, which allows users from one domain to access resources in another, creating a secure and authenticated connection between the domains.

#### Forest&#x20;

A forest is a collection of Active Directory domains. It is the topmost container and contains all of the AD objects introduced below, including but not limited to domains, users, groups, computers, and Group Policy objects. A forest can contain one or multiple domains and be thought of as a state in the US or a country within the EU. Each forest operates independently but may have various trust relationships with other forests. as we can see with the name that Forest is a collation of tree .

what is tree we gone se below

#### Tree

A tree is a collection of Active Directory domains that begins at a single root domain. A forest is a collection of AD trees. Each domain in a tree shares a boundary with the other domains. A parent-child trust relationship is formed when a domain is added under another domain in a tree. Two trees in the same forest cannot share a name (namespace). Let's say we have two trees in an AD forest: `udayispro.local` and `x4csispro.local`. A child domain of the first would be `crop.udayispro.local` while a child domain of the second could be `corp.x4csispro.local`. All domains in a tree share a standard Global Catalog which contains all information about objects that belong to the tree.

> What is the difference between tree and forest and domain?
>
> \
> The main difference between Tree and Forest in Active Directory is that a Tree is a collection of domains while the forest is a set of trees in an active directory. In brief, a tree is a collection of domains whereas a forest is a collection of trees

#### **Container**

container is a type of object used to organize other objects, such as users, groups, and computers

#### leaf

Leaf objects are AD objects that can not contain other objects within them. Computers, users, and printers are all examples of leaf objects.

#### **Global Unique Identifier (GUID)**

so to understand GUID first we have to know what is UID.

* "UID" can refer to a user identifier, though it's important to distinguish between different attributes that serve similar purposes.
* `uid` attribute , `uidNumber` attribute, `sAMAccountName` attribute and Security Identifier (SID)

these are some similar attributes which serve similar purposes.

A GUID in Active Directory is a Globally Unique Identifier stored in the `objectGUID` attribute, which serves as a permanent, 128-bit unique identifier for an object, like a user or group. This value is assigned when the object is created and never changes, even if the object is renamed or moved, making it a reliable identifier for applications.

&#x20;The GUID is stored in the `ObjectGUID` attribute. When querying for an AD object (such as a user, group, computer, domain, domain controller, etc.), we can query for its `objectGUID` value using PowerShell or search for it by specifying its distinguished name, GUID, SID, or SAM account name. GUIDs are used by AD to identify objects internally. Searching in Active Directory by GUID value is probably the most accurate and reliable way to find the exact object you are looking for, especially if the global catalog may contain similar matches for an object name. Specifying the `ObjectGUID` value when performing AD enumeration will ensure that we get the most accurate results pertaining to the object we are searching for information about

#### **Security principals**

A **security principal**are anything that the operating system can authenticate, including users, computer accounts, or even threads/processes that run in the context of a user or computer account and can be **assigned permissions** to access resources.&#x20;

**Types of Security Principals**

There are **three main types** of security principals in Active Directory:

| Type                  | Description                                                                                     | Examples                                      |
| --------------------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------- |
| **User Accounts**     | Represent individuals who log into the domain or services.                                      | `uday.pro`, `Administrator`, service accounts |
| **Computer Accounts** | Represent computers joined to the domain they authenticate to AD just like users.               | `PC01$`, `SRV-DC01$`                          |
| **Security Groups**   | Represent collections of users/computers. Permissions assigned to a group apply to all members. | `Domain Admins`, `HR Users`                   |

#### **Security Identifier (SID)**

**Security Identifier SID** is used as a unique identifier for a security principal or security group. Every account, group, or process has its own unique SID, which, in an AD environment, is issued by the domain controller and stored in a secure database. A SID can only be used once. Even if a user account is deleted, its SID is never reused to identify a new user .&#x20;

#### **Distinguished Name (DN)**

A  **Distinguished Name** describes the full path to an object in AD (such as `cn=uday, ou=IT, ou=Employees, dc=Extreme, dc=local`). In this example, the user `uday` works in the IT department of the company Extreme, and his account is created in an Organizational Unit (OU) that holds accounts for company employees. The Common Name (CN) `uday` is just one way the user object could be searched for or accessed within the domain.

#### **sAMAccountName**

`sAMAccountName` is the user logon name in Active Directory, designed for backward compatibility with older Windows versions It is a 20-character or less username that must be unique within its domain, used for authentication, and is sometimes referred to as the "pre-Windows 2000" or "NT4" logon name. While `UserPrincipalName` (UPN) is the more modern logon format, `sAMAccountName` is still used for down-level authentication and by various systems and applications

#### **User Principal Name(UPN)**

A User Principal Name (UPN) is an Internet-style login name for a user, formatted like an email address, such as `username@example.com`. It is composed of two parts: a user account name (prefix) and a domain name (suffix), joined by the "@" symbol. The UPN is used for authenticating users to a system and must be unique within a directory forest.  so unique as in that duplicate are not allowed&#x20;

lets get it by a example:-

In Active Directory, inside a **forest** (the entire AD environment that can contain multiple domains),\
each **UPN (User Principal Name)** must belong to **only one user**.

Imagine your company network has three domains:

* `india.company.com`
* `usa.company.com`
* `uk.company.com`

If one user already has this UPN:

```
rahul@company.com
```

then **no other user** in the entire forest not in India, USA, or UK can have the same `rahul@company.com` UPN

That’s because when someone tries to sign in as `rahul@company.com`, Active Directory must know **exactly which account** they are. If two accounts had the same UPN, AD wouldn’t know which one to authenticate.

#### **FSMO Roles**

In the early days of AD, if you had multiple DCs in an environment, they would fight over which DC gets to make changes, and sometimes changes would not be made properly. Microsoft then implemented "last writer wins," which could introduce its own problems if the last change breaks things. They then introduced a model in which a single "master" DC could apply changes to the domain while the others merely fulfilled authentication requests. This was a flawed design because if the master DC went down, no changes could be made to the environment until it was restored. To resolve this single point of failure model, Microsoft separated the various responsibilities that a DC can have into [Flexible Single Master Operation (FSMO)](https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/fsmo-roles) roles. These give Domain Controllers (DC) the ability to continue authenticating users and granting permissions without interruption (authorization and authentication). There are five FSMO roles: `Schema Master` and `Domain Naming Master` (one of each per forest), `Relative ID (RID) Master` (one per domain), `Primary Domain Controller (PDC) Emulator` (one per domain), and `Infrastructure Master` (one per domain). All five roles are assigned to the first DC in the forest root domain in a new AD forest. Each time a new domain is added to a forest, only the RID Master, PDC Emulator, and Infrastructure Master roles are assigned to the new domain. FSMO roles are typically set when domain controllers are created, but sysadmins can transfer these roles if needed. These roles help replication in AD to run smoothly and ensure that critical services are operating correctly. We will walk through each of these roles in detail later in this section.

> more notes are on the way this is not the end&#x20;

#### **Global Catalog**

**Global Catalog(GC)** is a domain controller that stores copies of ALL objects in an Active Directory forest. The GC stores a full copy of all objects in the current domain and a partial copy of objects that belong to other domains in the forest and it also allows users and applications to find objects in an Active Directory domain tree, given one or more attributes of the target object. GC is a feature that is enabled on a domain controller and performs the following functions:&#x20;

* **Authentication**: During the logon process in a multi-domain environment, the authenticating domain controller uses the GC to get a user principal name (UPN, `username@domain.com`) and find the user's account, even if it is in another domain.

#### **Read-Only Domain Controller (RODC)**

A **Read-Only Domain Controller (RODC)** is a type of domain controller that holds a non-writable copy of the Active Directory database.

#### **Service Principal Name (SPN)**

A **Service Principal Name (SPN)** is a unique identifier for a service instance that allows a client to use Kerberos authentication to connect to a service without knowing the specific service account, enabling mutual authentication. without knowing the name means allowing a client application to request the service to authenticate an account without needing to know the account name.

in Hinglish for batter explanation&#x20;

> SPN ek unique identifier hota hai jo kisi service instance ko represent karta hai na ke kisi insan ka username Jab client authenticate karna chahta hai to woh account name ki jagah SPN ke liye Kerberos ticket maangta hai KDC SPN ko dekh kar pata laga leta hai ke woh SPN kaunsa service account (ya computer account / managed service account) ke saath registered hai aur phir usi account ki secret key ko use karke service ticket issue karta hai Isliye client ko actual account name nahi pta hona chahiye SPN hi kaafi hota hai.

#### **Group Policy Object(GPO)**

A **GPO (Group Policy Object)** is a **set of rules or settings** that an administrator creates to **control how computers and user accounts behave** in a Windows network that uses **Active Directory**.

**Think of it like this:**

If you manage many computers in an organization you don’t want to configure each one manually.\
Instead, you create a **GPO** that says things like:

* Hide the Control Panel for all users.
* Set the same wallpaper on every PC.
* Force users to change their passwords every 90 days.
* Install Microsoft Office automatically on all computers.

That GPO is then **automatically applied** to all the computers or users you choose (in an OU, domain, or site).

#### **Access Control List (ACL)**

An **Access Control List (ACL)** is the ordered collection of Access Control Entries (ACEs) that apply to an object.

**Real-life analogy**

Think of your **house**:

* **ACL** = The list on your door of who can enter and what they can do inside.

```
used chat-gpt for bellow
```

Here’s a clearer visual explanation of **ACLs in BloodHound** with inbound and outbound permissions:

```
          [Alice]          [Bob]          [Charlie]
             |              |               |
             | Modify       |               |
             |------------->|               |
             |  (Inbound)   |               |
             |              | Read          |
             |              |-------------> |
             |              |  (Outbound)   |

Object: Bob
-------------------------
Inbound ACLs (who can act on Bob):
  - Alice: Modify
  - Charlie: None

Outbound ACLs (who Bob can act on):
  - Bob -> Charlie: Read
  - Bob -> Alice: None
```

**Explanation:**

* **Inbound ACEs**: Show permissions **coming into Bob**, i.e., who can affect Bob. Here, Alice can modify Bob.
* **Outbound ACEs**: Show permissions **going out from Bob**, i.e., who Bob can affect. Here, Bob can read Charlie.

**Mnemonic:**

* **Inbound = who can touch me**
* **Outbound** = **who I can touch**

An **ACL (Access Control List)** just contains a **list of permissions (ACEs)** on an object. It doesn’t inherently show inbound or outbound it’s just rules.

#### **Access Control Entries (ACEs)**

Each **Access Control Entries (ACEs)** in an ACL identifies a trustee (user account, group account, or logon session) and lists the access rights that are allowed, denied, or audited for the given trustee.

ACL = list of rules applied on an object, and ACE = a single rule in that list.

#### **Discretionary Access Control List (DACL)**

A Discretionary Access Control List (DACL) is a special type of ACL that contains multiple rules (ACEs) specifically defining allow and deny permissions. Its purpose is to determine who can or cannot access an object, such as a file, folder, or Active Directory object. A DACL only includes those ACEs that specify allow or deny permissions, for example, Alice -> Read -> Allow or Bob  Write -> Deny. If a DACL is empty, no one can access the object. A DACL is used solely for access control, not for auditing; auditing is handled by a System ACL (SACL). In short, a DACL defines who can or cannot access this object, while a SACL defines which actions should be audited.

**in Hinglish for batter explanation**&#x20;

> Discretionary Access Control List (DACL) ek special type ka ACL hai, jisme multiple rules (ACEs) hote hain jo specifically allow aur deny permissions define karte hain. Iska purpose ye decide karna hai ki kaun kisi object, jaise file, folder, ya Active Directory object, ko access kar sakta hai aur kaun nahi. DACL me sirf un ACEs ko include kiya jata hai jo allow ya deny permissions specify karte hain, jaise Alice -> Read -> Allow ya Bob -> Write -> Deny. Agar DACL empty ho to object ko koi bhi access nahi kar sakta. DACL sirf access control ke liye hoti hai, audit ke liye nahi; audit ke liye System ACL (SACL) use ki jati hai. Yaad rakhne ke liye: DACL batata hai who can or cannot access this object aur SACL batata hai which actions should be audited.

#### **System Access Control Lists (SACL)**

A System Access Control List (SACL) is a special type of ACL that determines which actions should be audited (recorded). It does not grant or deny access; it is used solely for security auditing. A SACL contains ACEs (Access Control Entries) that specify which user or group’s actions should be logged and whether to log them on success, failure, or both. For example, if Alice successfully reads a file, or if Bob fails to delete a file, these actions would be recorded according to the SACL rules. The key difference between DACL and SACL is that a DACL controls access (who can or cannot access an object), whereas a SACL controls auditing (which actions should be logged).

**in Hinglish for batter explanation**&#x20;

> SACL (System Access Control List) ek special type ka ACL hai jo decide karta hai ki kaunse actions audit (record) kiye jayenge. Ye access allow ya deny nahi karta, sirf security auditing ke liye hota hai. SACL me ACEs (Access Control Entries) hote hain jo specify karte hain ki kaunse user ya group kaunsa action perform karega aur us action ko record karna hai, chahe action success ho ya failure. Example ke liye, agar Alice file read kare aur success ho, ya Bob file delete karne ki koshish fail ho, to ye dono actions SACL ke rules ke according event log me record honge. DACL aur SACL me fark ye hai ki DACL access control ke liye hota hai (who can/cannot access), aur SACL auditing ke liye (which actions should be logged).

#### **Fully Qualified Domain Name (FQDN)**

A **Fully Qualified Domain Name (FQDN**) is a complete domain name that uniquely identifies a host or machine on a network or the Internet. It includes the hostname, domain name, and top-level domain (TLD) = Top-Level Domain. For example, in `mail.google.com`, `mail` is the hostname, `google` is the domain, and `com` is the TLD. FQDN is also used in Active Directory, such as `corp.example.com`, to identify a domain. The purpose of an FQDN is solely to locate a host uniquely; it does not directly relate to ACLs, DACLs, SACLs, or ACEs, which are used for permissions and auditing.

**in Hinglish for batter explanation**&#x20;

> **FQDN (Fully Qualified Domain Name) ek complete domain name hota hai jo ek host ya machine ko network ya Internet me uniquely identify karta hai. Isme hostname, domain name, aur top-level domain (TLD) include hota hai, jaise `mail.google.com` me `mail` hostname hai, `google` domain hai, aur `com` TLD hai. AD (Active Directory) me bhi FQDN use hota hai, jaise `corp.example.com` domain ko identify karne ke liye. FQDN ka kaam sirf host ko uniquely locate karna hai; iska ACL, DACL, SACL ya ACE se direct relation nahi hai, ye permissions aur auditing rules ke liye use nahi hota.**

#### **Active Directory Users and Computers (ADUC)**

ADUC is a GUI console commonly used for managing users, groups, computers, and contacts in AD. Changes made in ADUC can be done via PowerShell as well.

## Trust

A trust is used to establish `forest-forest` or `domain-domain` authentication, allowing users to access resources in (or administer) another domain outside of the domain their account resides in. A trust creates a link between the authentication systems of two domains.

There are several trust types.

| **Trust Type** | **Description**                                                                                                                                                        |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Parent-child` | Domains within the same forest. The child domain has a two-way transitive trust with the parent domain.                                                                |
| `Cross-link`   | a trust between child domains to speed up authentication.                                                                                                              |
| `External`     | A non-transitive trust between two separate domains in separate forests which are not already joined by a forest trust. This type of trust utilizes SID filtering.     |
| `Tree-root`    | a two-way transitive trust between a forest root domain and a new tree root domain. They are created by design when you set up a new tree root domain within a forest. |
| `Forest`       | a transitive trust between two forest root domains.                                                                                                                    |

**Trust Example**

![Diagram of Inlanefreight Forest showing trust types: Parent-child, Cross-link, External, Tree-root, and Forest. Domains include inlanefreight, corp.inlanefreight, wh.corp.inlanefreight, dev.inlanefreight, freightlogistics, and shippinglanes.](https://academy.hackthebox.com/storage/modules/74/trusts-diagram.png)

Trusts can be transitive or non-transitive.

* A transitive trust means that trust is extended to objects that the child domain trusts.
* In a non-transitive trust, only the child domain itself is trusted.

Trusts can be set up to be one-way or two-way (bidirectional).

* In bidirectional trusts, users from both trusting domains can access resources.
* In a one-way trust, only users in a trusted domain can access resources in a trusting domain, not vice-versa. The direction of trust is opposite to the direction of access.









