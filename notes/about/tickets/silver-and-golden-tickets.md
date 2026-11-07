---
icon: tickets-simple
---

# Silver & Golden Tickets

### Introduction

This guide explains how Kerberos authentication works in Active Directory environments, focusing on three critical concepts: PAC (Privilege Attribute Certificate), Silver Tickets, and Golden Tickets. By the end, you'll understand what these are, how they work, and how attackers exploit them.

***

### 1. PAC (Privilege Attribute Certificate)

#### What is PAC?

Think of PAC as your **digital security badge** in Active Directory. Just like a physical badge shows security guards who you are and which doors you can open, PAC tells Windows services what permissions you have.

#### What Information Does PAC Contain?

PAC is packed with information about you:

* **Username** - Your account name (e.g., "pixis")
* **User ID (RID)** - Your unique identifier number
* **Group Memberships** - All groups you belong to (Domain Users, Domain Admins, etc.)
* **Security Flags** - Account settings (password expiry, login restrictions, etc.)
* **Timestamps** - When you logged in, password last changed, etc.
* **Domain Info** - Which domain you belong to

#### Why PAC Matters

PAC is included in every Kerberos ticket you receive. When you try to access a file share, run a command, or use any service, that service reads your PAC to decide what you're allowed to do.

**Important:** PAC is encrypted and digitally signed by the domain controller, so you can't just edit it and give yourself admin rights. It's tamper-proof from the user's perspective.

***

### 2. Silver Ticket Attack

#### What is a Silver Ticket?

A Silver Ticket is a **forged service ticket (TGS)** that an attacker creates themselves instead of getting it from the domain controller. It's like making your own fake ID card to get into a specific building.

#### How Kerberos Normally Works (The Legit Way)

Let me explain the normal process first so you understand what attackers are bypassing:

1. **You want to access a service** - Let's say you want to open `\\SERVER01\share`
2. **You ask the KDC** - Your computer contacts the Key Distribution Center (domain controller) and says "I need access to CIFS on SERVER01"
3. **KDC creates a ticket** - The KDC creates a TGS (service ticket) that contains:
   * Your username
   * Your PAC (with all your permissions)
   * A session key for encryption
4. **KDC encrypts the ticket** - This ticket is encrypted using SERVER01's password hash (the machine account password)
5. **You send the ticket to the service** - Your computer sends this encrypted ticket to SERVER01
6. **Service decrypts and verifies** - SERVER01 decrypts it with its own password, checks your permissions in the PAC, and grants or denies access

#### How Silver Ticket Attack Works

Now here's what an attacker does:

**Step 1: Compromise the Service Account**

First, the attacker needs to steal the password hash of the service account. For example, if they want to access DESKTOP-01, they need the NT hash of the computer account `DESKTOP-01$`.

How do they get this? Usually by:

* Extracting it from memory on the compromised machine
* DCSync attacks
* Credential dumping tools like Mimikatz

**Step 2: Forge the Ticket**

Once they have the service account's NT hash, the attacker can create their own fake TGS ticket. They build a ticket that contains:

* **Domain name** - `adsec.local`
* **Service name (SPN)** - `CIFS/DESKTOP-01.adsec.local`
* **Username** - They can choose ANY username, even fake ones like "pixisAdmin"
* **Fake PAC** - Here's the magic: they create a PAC claiming they're Domain Admin
* **Session key** - They pick their own random session key
* **Timestamps** - They can set validity dates (even years into the future!)

**Step 3: Encrypt with the Stolen Hash**

They encrypt this forged ticket using the NT hash they stole. Since the service account expects tickets encrypted with its own password, the fake ticket looks legitimate to the service.

**Step 4: Access the Service**

The attacker sends this forged ticket directly to DESKTOP-01 (bypassing the KDC entirely). The service:

* Decrypts the ticket successfully (because it was encrypted with the correct hash)
* Reads the PAC and sees "Domain Admin" privileges
* Grants full access to the attacker

#### Why Does This Work?

Here's the critical part: PAC has **two signatures**:

1. **First signature** - Created with the service account's secret
2. **Second signature** - Created with the domain controller's secret (krbtgt account)

Most services only verify the first signature! They don't check if the domain controller actually issued this ticket. This is especially true for services running as SYSTEM or accounts with special privileges.

So even though the attacker can't forge the second signature (they don't have krbtgt hash), it doesn't matter because the service doesn't check it.

#### Commands to Create Silver Tickets

**Windows (Using Mimikatz)**

```
kerberos::golden /domain:adsec.local /user:random_user /sid:S-1-5-21-1423455951-1752654185-1824483205 /rc4:0123456789abcdef0123456789abcdef /target:DESKTOP-01.adsec.local /service:cifs /ptt
```

**Parameter breakdown:**

* `/domain` - Target domain name
* `/user` - Username (can be fake, doesn't need to exist)
* `/sid` - Domain SID (Security Identifier)
* `/rc4` - NT hash of the service account (DESKTOP-01$ in this example)
* `/target` - Server you want to access
* `/service` - Service type (cifs for file shares, http for web, ldap, etc.)
* `/ptt` - Pass-the-ticket (inject directly into current session)

**Linux (Using Impacket)**

**Step 1: Create the ticket**

```bash
ticketer.py -nthash 0123456789abcdef0123456789abcdef -domain-sid S-1-5-21-1423455951-1752654185-1824483205 -domain adsec.local -spn CIFS/DESKTOP-01.adsec.local random_user
```

**Step 2: Export to environment variable**

```bash
export KRB5CCNAME='/path/to/random_user.ccache'
```

**Step 3: Use the ticket with Impacket tools**

```bash
psexec.py -k DESKTOP-01.adsec.local
```

The `-k` flag tells the tool to use Kerberos authentication with the ticket you just created.

***

### 3. Golden Ticket Attack

#### What is a Golden Ticket?

A Golden Ticket is a **forged TGT (Ticket Granting Ticket)** - the master ticket that lets you request access to ANY service in the domain. If a Silver Ticket is a fake key to one room, a Golden Ticket is a fake master key to the entire building.

#### The Difference Between TGT and TGS

Let me clarify this because it's important:

* **TGT (Ticket Granting Ticket)** - The initial ticket you get when you log in. This ticket lets you request service tickets. It's encrypted with the krbtgt account's password.
* **TGS (Ticket Granting Service)** - A service-specific ticket. You use your TGT to request these from the KDC. Each TGS is encrypted with that service's password.

**Normal flow:**

1. User logs in -> Gets TGT
2. User needs file share -> Uses TGT to request TGS for CIFS
3. User needs SQL database -> Uses TGT to request TGS for SQL
4. And so on...

#### How Golden Ticket Attack Works

**Step 1: Compromise the krbtgt Account**

The krbtgt account is a special account in Active Directory. Its password hash is used to encrypt all TGT tickets. To create a Golden Ticket, attackers need this hash.

How do they get it? Usually by:

* Dumping it from domain controller memory
* DCSync attack (impersonating a domain controller)
* Backing up NTDS.dit (Active Directory database) and extracting it

**Step 2: Forge a TGT**

With the krbtgt hash, attackers create a completely fake TGT containing:

* **Domain name** - `adsec.local`
* **Username** - ANY username (can be fake like "TotallyLegitAdmin")
* **Fake PAC** - Claims to be Domain Admin, Enterprise Admin, or any group
* **Session key** - Attacker chooses this
* **Validity period** - Can set it for 10 years if they want!

**Step 3: Use the Golden Ticket**

Here's what makes it "golden" - with this forged TGT, the attacker can:

1. Request legitimate TGS tickets from the KDC for ANY service
2. The KDC accepts the TGT because it's encrypted correctly
3. The KDC issues real service tickets based on the fake permissions in the PAC
4. Attacker gets access to everything: file shares, databases, domain controllers, etc.

#### Why This is So Powerful

Unlike Silver Tickets that bypass the KDC, Golden Tickets **use the KDC normally**. The domain controller thinks it's issuing tickets to a legitimate Domain Admin because:

* The TGT is encrypted with the correct key
* The PAC says they're Domain Admin
* The signatures check out

From the KDC's perspective, everything looks normal!

#### Commands to Create Golden Tickets

**Windows (Using Mimikatz)**

```
kerberos::golden /domain:adsec.local /user:random_user /sid:S-1-5-21-1423455951-1752654185-1824483205 /krbtgt:0123456789abcdef0123456789abcdef /ptt
```

**Parameter breakdown:**

* `/domain` - Your domain name
* `/user` - Any username (doesn't have to exist)
* `/sid` - Domain SID
* `/krbtgt` - NT hash of the krbtgt account (THE GOLDEN KEY!)
* `/ptt` - Inject ticket into current session

**Notice:** We don't specify a target or service because this ticket works for everything.

**Linux (Using Impacket)**

**Step 1: Create the Golden Ticket**

```bash
ticketer.py -nthash 0123456789abcdef0123456789abcdef -domain-sid S-1-5-21-1423455951-1752654185-1824483205 -domain adsec.local random_user
```

**Step 2: Export to environment**

```bash
export KRB5CCNAME='/path/to/random_user.ccache'
```

**Step 3: Use it for ANYTHING**

```bash
# Dump all domain credentials
secretsdump.py -k DC-01.adsec.local -just-dc-ntlm

# Remote command execution on any machine
psexec.py -k WORKSTATION-05.adsec.local

# Access any file share
smbclient.py -k //SERVER/C$
```

The possibilities are endless with a Golden Ticket!

***

### Quick Comparison Table

| Feature                     | Silver Ticket                    | Golden Ticket                |
| --------------------------- | -------------------------------- | ---------------------------- |
| **What's forged**           | TGS (Service Ticket)             | TGT (Ticket Granting Ticket) |
| **Required hash**           | Service account NT hash          | krbtgt account NT hash       |
| **Access scope**            | One specific service only        | Entire domain (all services) |
| **Talks to KDC**            | No (bypasses it)                 | Yes (uses it normally)       |
| **Detection difficulty**    | Harder (no KDC logs)             | Easier (generates KDC logs)  |
| **Persistence**             | Until service password changes   | Until krbtgt reset twice     |
| **Compromise level needed** | Service/machine account          | Domain Admin or DC access    |
| **Flexibility**             | Need separate ticket per service | One ticket for everything    |

***

### Key Takeaways

**PAC (Privilege Attribute Certificate):**

* Your digital security badge containing all your permissions and group memberships
* Included in every Kerberos ticket
* Encrypted and signed to prevent tampering

**Silver Ticket:**

* Forged service ticket for ONE specific service
* Requires the service account's NT hash
* Bypasses the domain controller completely
* Harder to detect but limited in scope

**Golden Ticket:**

* Forged TGT that works for the ENTIRE domain
* Requires the krbtgt account's NT hash
* Uses the domain controller normally
* Ultimate persistence and access

**Bottom line:** These attacks show why protecting password hashes is critical. Once attackers have these hashes, they can create tickets that look completely legitimate to the system, even if they're totally fake. Defense requires multiple layers: good monitoring, quick incident response, and limiting the blast radius of any single compromise.

***

### References

1. [https://en.hackndo.com/kerberos-silver-golden-tickets/](https://en.hackndo.com/kerberos-silver-golden-tickets/)
