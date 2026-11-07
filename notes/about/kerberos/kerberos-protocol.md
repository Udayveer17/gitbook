---
icon: paw
---

# Kerberos Protocol

### What Even is Kerberos?

Alright, so Kerberos is basically THE authentication protocol used in Windows networks and Active Directory. It's how your computer proves who you are without constantly asking for your password every single time you want to access a file share, printer, or any service on the network.

The cool thing about Kerberos? **Your password never gets sent over the network.** Instead, it uses a ticket-based system. Think of it like a movie theater - you buy a ticket once, and then you just show that ticket to get in. No need to pay again and again.

### Why Should You Care?

If you're into pentesting, red teaming, or just understanding how Windows networks work, Kerberos is absolutely critical. It's everywhere in Active Directory environments, and understanding it means you can:

* Understand common AD attacks (Kerberoasting, Pass-the-Ticket, Golden Tickets, etc.)
* Know how authentication actually works
* Spot misconfigurations and weaknesses

Plus, honestly, once you get it, it's actually pretty elegant.

### The Big Players in Kerberos

Before we dive in, you need to know about three main entities:

**1. The Client** - That's you, the user trying to access stuff

**2. The Service** - Whatever you're trying to use (file share, SQL database, web app, etc.)

**3. The Key Distribution Center (KDC)** - This is the Domain Controller in Active Directory. It's the trusted middleman that knows everyone's secrets

Here's the important part: The KDC knows EVERYONE's password hash - users, computers, services, everything. But nobody else knows each other's passwords. Only you know your password, and only the KDC knows everyone else's.

### The Party Analogy (Best Way to Understand This)

I love this analogy because it makes everything click:

Imagine you're going to a party:

1. **You show your ID at the door** - This proves you are who you say you are (this is like getting your TGT)
2. **You go to the cashier with your ID** - You want a drink, so you exchange your ID for a drink ticket (this is like getting a Service Ticket/TGS)
3. **You go to the bar with the drink ticket** - The bartender sees the official stamp, knows it's legit, and serves you a beer (this is using the service)

The bartender doesn't need to know your password or check your ID - they just trust the ticket from the cashier. That's basically Kerberos.

### How Kerberos Actually Works - The Three Steps

The whole process has three main phases:

#### Step 1: Authentication Service (AS) - Getting Your TGT

This is where you prove who you are to the Domain Controller.

**What You Send (KRB\_AS\_REQ):**

* Your username
* Current timestamp, encrypted with your password hash

**Why the timestamp?** Because the DC needs to verify you actually know the password. You encrypt the current time with your password hash, and the DC tries to decrypt it. If it works, you're legit.

**What You Get Back (KRB\_AS\_REP):**

* A **session key** (encrypted with your password hash)
* A **TGT - Ticket Granting Ticket** (encrypted with the KDC's secret key)

The TGT contains:

* Your username
* Your session key
* Validity period (these tickets expire!)
* Your PAC (Privilege Attribute Certificate) - this has your SID and all your group memberships

**Important:** The TGT is encrypted with the KDC's key, so only the KDC can read it. You can't even read your own TGT. But you can use it.

#### Step 2: Ticket-Granting Service (TGS) - Getting Access to Specific Services

Now you want to use a specific service, let's say access a file share (CIFS) on SERVER01.

**What You Send (KRB\_TGS\_REQ):**

* Your TGT (encrypted with KDC's key)
* The service you want: `CIFS/SERVER01`
* An **authenticator** (your username + timestamp, encrypted with the session key)

**Why the authenticator?** This proves you're not just someone who stole the TGT. You need to know the session key to create a valid authenticator.

The KDC does this verification:

1. Decrypts the TGT (only it can do this)
2. Extracts the session key from the TGT
3. Uses that session key to decrypt the authenticator
4. Compares the info in both - if they match, you're legit

**What You Get Back (KRB\_TGS\_REP):**

* A **Service Ticket** (encrypted with the service's secret key)
* A **new session key** for communicating with the service

Both of these are wrapped in encryption using your original session key.

The Service Ticket (also called TGS) contains:

* Service name (CIFS/SERVER01)
* Your username
* Your PAC
* The new session key for you and the service

#### Step 3: Application Request (AP) - Actually Using the Service

Finally, you go to the actual service with your ticket.

**What You Send (KRB\_AP\_REQ):**

* The Service Ticket (encrypted with the service's key)
* A new authenticator (encrypted with the service session key)

**What Happens:**

1. The service decrypts the ticket with its own secret key
2. Extracts the session key from the ticket
3. Uses that session key to decrypt your authenticator
4. Verifies everything matches
5. Boom - you're in!

The service trusts this because:

* Only the KDC knows the service's secret
* So only the KDC could have created this ticket
* And you proved you have the session key

### Why This Design is Brilliant

Think about what just happened:

* **Your password was never sent over the network** - Only encrypted timestamps
* **Services never see your password** - They just trust the KDC
* **Everything is time-limited** - Tickets expire, reducing risk
* **Mutual authentication** - Both sides verify each other
* **Centralized management** - The KDC controls everything

This is why Kerberos is considered one of the most secure authentication protocols out there.

### The Security Features

**Symmetric Key Cryptography:** Kerberos uses the same key to encrypt and decrypt. Your password hash becomes your encryption key.

**No Passwords on the Network:** Seriously, your actual password never travels. Even if someone captures all your traffic, they won't see your password.

**Time-Limited Tickets:** Everything expires. Even if someone steals a ticket, it's only good for a limited time (usually 10 hours for TGTs, less for service tickets).

**The Authenticator:** This is genius - it proves you're not just replaying old traffic. You create a fresh authenticator each time using the session key and current timestamp.

### Common Misconceptions

**"The TGT is super secret"** Nope! The TGT is considered public information. You can't read it anyway (it's encrypted with the KDC's key). The important part is the session key.

**"Kerberos is unbreakable"** Not quite. While the protocol itself is solid, there are attacks:

* Kerberoasting (cracking service account passwords)
* AS\_REP Roasting (accounts without pre-auth)
* Pass-the-Ticket (stealing tickets from memory)
* Golden/Silver Tickets (forging tickets)
* Brute force weak passwords

Most attacks exploit misconfigurations or weak passwords, not the protocol itself.

### Real-World Example

Let's say I'm user "pixis" and I want to access a file share:

1. **Morning login:**
   * I type my password once
   * My computer encrypts a timestamp and sends it to the DC
   * DC verifies and sends me back a TGT
   * Now I'm "authenticated" for the next 10 hours
2. **Opening a file share:**
   * My computer sends the TGT to the DC and says "I need CIFS/FILESERVER"
   * DC sends back a service ticket
   * My computer presents this ticket to FILESERVER
   * FILESERVER lets me in
3. **Later, accessing SQL database:**
   * Same TGT, different service request
   * Get a new service ticket for SQL
   * Access granted

All of this happens automatically. I only typed my password once at the beginning.

### What Can Go Wrong?

**Weak Passwords:** If your password is "Password123", someone can crack your tickets offline. Use strong passwords!

**Unconstrained Delegation:** Lets computers store your TGT - dangerous if that computer is compromised.

**Service Accounts with SPNs:** These can be Kerberoasted if they have weak passwords.

**No Pre-Authentication:** Some accounts don't require pre-auth - these can be AS\_REP Roasted.

**Outdated Encryption:** Some orgs still use RC4 instead of AES for Kerberos. RC4 is weaker.

### Tools for Testing Kerberos

If you're pentesting:

**Rubeus** - Swiss army knife for Kerberos attacks **Impacket** - Python tools for Kerberos operations **Mimikatz** - Can extract tickets from memory **BloodHound** - Maps out Kerberos relationships in AD

### Defense Tips

If you're defending:

* **Monitor for unusual Kerberos traffic** - Especially TGS requests for high-value accounts
* **Use strong passwords** - Especially for service accounts
* **Enable pre-authentication** - Don't disable it unless absolutely necessary
* **Use AES encryption** - Not RC4
* **Limit delegation** - Be very careful with unconstrained delegation
* **Audit SPNs** - Know which accounts have Service Principal Names

### Final Thoughts

Kerberos might seem complicated at first, but once you break it down:

1. Get authenticated -> Receive TGT
2. Request service access -> Receive Service Ticket
3. Use the service -> Present Service Ticket

Each step builds trust without ever exposing your password.

Understanding Kerberos is absolutely essential for anyone working with Active Directory - whether you're attacking it or defending it. The protocol itself is solid, but human mistakes (weak passwords, misconfigurations) create the vulnerabilities we exploit.

***

**TL;DR:** Kerberos is a ticket-based authentication system that lets you prove who you are without sending passwords over the network. You get a TGT from the DC, exchange it for Service Tickets, and use those to access stuff. It's secure by design, but can be attacked through misconfigurations or weak passwords.What is Kerberos?
