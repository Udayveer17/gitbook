---
icon: server
---

# AS\_REP Roasting

## AS\_REP Roasting - A Simple Breakdown

### What's This All About?

So basically, AS\_REP Roasting is a pretty cool attack technique in Active Directory environments. It lets you grab password hashes for certain user accounts without even needing to authenticate first. Sounds crazy, right? Let me break it down.

### How Kerberos Normally Works

Usually when you want to get a TGT (Ticket Granting Ticket) from the domain controller, here's what happens:

1. You send a request: "Hey, I'm Bob, give me a TGT"
2. BUT - you also have to prove you're actually Bob by sending an authenticator encrypted with Bob's password
3. The DC checks it and says "Cool, here's your TGT"

The DC sends back a KRB\_AS\_REP response which contains:

* The TGT (encrypted with the DC's secret)
* A session key (encrypted with YOUR password)

This is important - that session key part is encrypted with your password. So if anyone could just ask for a TGT without proving who they are, they could grab that encrypted session key and crack it offline to get the password.

**That's why pre-authentication exists** - to stop this exact attack.

### The Vulnerability

Here's where it gets interesting. For some weird reasons (honestly, even security researchers don't fully understand why admins do this), you can actually DISABLE pre-authentication for specific user accounts.

I've seen some cases where:

* Legacy systems need it disabled
* Some Unix database integrations require it
* Old applications that don't support modern Kerberos

When pre-authentication is disabled for an account, anyone can request a TGT for that user WITHOUT proving they own the account. The DC will just hand over the KRB\_AS\_REP response.

### The Attack

Here's how AS\_REP Roasting works:

**Step 1: Find vulnerable accounts**

* Look for accounts with "Do not require Kerberos preauthentication" enabled
* You can enumerate these if you have domain access

**Step 2: Request the TGT**

* Send a KRB\_AS\_REQ for the target account
* No authentication needed!
* DC sends back KRB\_AS\_REP with the encrypted session key

**Step 3: Crack offline**

* Take that encrypted part of the response
* Use tools like John the Ripper or Hashcat
* Brute force the password offline at your own pace

### Tools You Can Use

**Rubeus** - Modern and popular

```
Rubeus.exe asreproast
```

**Impacket's GetNPUsers.py** - Great for Linux

```
GetNPUsers.py domain.com/username -no-pass
```

**ASREPRoast** - The OG tool by Harmj0y

```
nxc ldap 0.0.0.0 -u user -p '' --asreproast output.txt
```

**NetExec** -&#x20;

### Real World Scenario

Let's say you're doing a pentest and you don't have any credentials yet. You could:

1. Do some OSINT to find valid usernames
2. Check each username to see if pre-auth is disabled
3. For any vulnerable accounts, grab the AS\_REP
4. Crack it offline

The beauty is you don't need ANY foothold in the network. Just need to be able to talk to the DC.

### How to Crack

Once you have the hash, you can use:

**John the Ripper:**

```
john --wordlist=rockyou.txt --format=krb5asrep hash.txt
```

**Hashcat:**

```
hashcat -m 18200 hash.txt wordlist.txt
```

### Defense

If you're a blue teamer:

* **Audit your AD** - Find accounts with pre-auth disabled
* **Only disable when absolutely necessary** - And document why
* **Use strong passwords** - For accounts that must have pre-auth disabled
* **Monitor for AS\_REP requests** - Unusual patterns might indicate roasting

### Final Thoughts

AS\_REP Roasting is honestly not super common in the wild because most admins don't disable pre-authentication. But when you find it, it's gold - especially if you're at the early stages of a pentest with no credentials.

It's similar to Kerberoasting but you don't even need a domain account to pull it off. That's what makes it powerful.

The moral of the story? Don't disable security features unless you absolutely have to, and if you do, make sure that account has a crazy strong password.
