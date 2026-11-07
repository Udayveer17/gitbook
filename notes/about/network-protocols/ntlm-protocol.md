---
icon: square-phone-hangup
---

# NTLM Protocol

## NTLM Protocol

The NTLM protocol is an authentication protocol used in Microsoft environments. In particular, it allows a user to prove their identity to a server in order to use a service offered by this server.

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

There are two possible scenarios:

* Either the user uses the credentials of a local account of the server, in which case the server has the user’s secret in its local database and will be able to authenticate the user;
* Or in an Active Directory environment, the user uses a domain account during authentication, in which case the server will have to ask the domain controller to verify the information provided by the user.

In both cases, authentication begins with a **challenge and response** between the client and the server.

#### Challenge - Response

The challenge/response principle is used so that the server verifies that the user knows the secret of the account he is authenticating with, without passing the password through the network. There are three steps in this exchange:

1. **Negotiation** : The client tells the server that it wants to authenticate to it. if want to know in deep you can read this -> [NEGOTIATE\_MESSAGE](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-nlmp/b34032e5-3aae-4bc6-84c3-c6d80eadf7f2).
2. **Challenge** : The server sends a challenge to the client. This is nothing more than a 64-bit random value that changes with each authentication request. if want to know in deep you can read this -> [CHALLENGE\_MESSAGE](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-nlmp/801a4681-8809-4be9-ab0d-61dcfe762786).
3. **Response** : The client encrypts the previously received challenge using a hashed version of its password as the key, and returns this encrypted version to the server, along with its username and possibly its domain. if want to know in deep you can read this -> [AUTHENTICATE\_MESSAGE](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-nlmp/033d32cc-88f9-4483-9bf2-b273055038ce).

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

#### Authentication secret

We said that the client uses a hashed version of their password as a key for the following reason: To avoid storing user passwords in clear text on the server. It’s the password’s hash that is stored instead. This hash is now the **NT hash**, which is nothing but the result of the [MD4](https://fr.wikipedia.org/wiki/MD4) function, **without salt**, nothing.

```
NThash = MD4(password)
```

So to summarize, when the client authenticates, it uses the MD4 fingerprint of its password to encrypt the challenge. Let’s then see what happens on the server side, once this response is received.

### Authentication

As explained earlier, there are two different scenarios. The first is that the account used for authentication is a local account, so the server has knowledge of this account, and it has a copy of the account’s secret. The second is that a domain account is used, in which case the server has no knowledge of this account or its secret. It will have to delegate authentication to the domain controller.

### SAM (Security Accounts Manager)

the server needs to store the local users and the hash of their password. The name of this database is the **SAM** (Security Accounts Manager). The SAM can be found in the registry, especially with the `regedit` tool, but only when accessed as **SYSTEM**. It can be opened as **SYSTEM** with [psexec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec):

```
psexec.exe -i -s regedit.exe
```

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

A copy is also on disk in `C:\Windows\System32\SAM`.

So it contains the list of local users and their hashed password, as well as the list of local groups. Well, to be more precise, it contains an encrypted version of the hashes. But as all the information needed to decrypt them is also in the registry (SAM and SYSTEM), we can safely say that their hashed version is stored there. If you want to see how the decryption mechanism works, you can go check [secretsdump.py code](https://github.com/SecureAuthCorp/impacket/blob/master/impacket/examples/secretsdump.py#L1124) or [Mimikatz code](https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/kuhl_m_lsadump.c).

SAM and SYSTEM databases can be backed up to extract the user’s hashed passwords database.

First we save the two databases in a file

```
reg.exe save hklm\sam save.save
reg.exe save hklm\system system.save
```

Then, we can use [secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py) to extract these hashes

```
secretsdump.py -sam sam.save -system system.save LOCAL
```

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

So to summarize, here’s the verification process.

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

Since the server sends a challenge (**1**) and the client encrypts this challenge with the hash of its secret and then sends it back to the server with its username (**2**), the server will look for the hash of the user’s password in its SAM database (**3**). Once it has it, it will also encrypt the challenge previously sent with this hash (**4**), and compare its result with the one returned by the user. If it is the same (**5**) then the user is authenticated! Otherwise, the user has not provided the correct secret.

#### Domain account

When a **domain account** authenticates using NTLM, the user’s password hash (NT hash) is not stored on the local server but on the **Domain Controller (DC)** inside the `NTDS.DIT` database. The process begins when the **server** sends a random challenge to the **client**. The client then encrypts this challenge with its NT hash (derived from the user’s password) and sends the result, along with its username and domain name, back to the server. Since the server doesn’t have the user’s NT hash, it can’t verify the response itself. Instead, it uses the **Netlogon service** to establish a **Secure Channel** with the Domain Controller a trusted, encrypted connection based on the server’s own machine account password. Through this channel, the server sends the user’s name, the original challenge, and the client’s response to the DC. The Domain Controller retrieves the user’s NT hash from the `NTDS.DIT` file, uses it to compute what the correct response should be, and compares it to the one received from the client. If they match, the DC confirms the user is authenticated and notifies the server through the secure channel, sending back additional user information like groups and privileges. If they don’t match, authentication fails. In short, for **local accounts**, verification is done by the local server using the SAM database; for **domain accounts**, it’s done by the Domain Controller via the Netlogon secure channel.

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

Same as before, the server sends a challenge (**1**) and the client **jsnow** encrypts this challenge with the hash of its secret and sends it back to the server along with its username and the domain name (**2**). This time the server will send this information to the domain controller in a **Secure Channel** using the **Netlogon** service (**3**). Once in possession of this information, the domain controller will also encrypt the challenge using the user’s hash, found in its NTDS.DIT database (**4**), and will then be able to compare its result with the one returned by the user. If it is the same (**5**) then the user is authenticated. Otherwise, the user has not provided the right secret. In both cases, the domain controller transmits the information to the server (**6**).

```
from chat gpt What is Netlogon
```

#### What is Netlogon?

**Netlogon** is a **Windows service and network protocol** used to **authenticate users and computers** within a Windows domain. It acts as the “communication bridge” between a **member computer or server** and the **Domain Controller (DC)** during authentication processes such as NTLM and Kerberos.

When a computer or server is part of a domain, it has its own **machine account** (like `SERVER01$`) and password known only to it and the Domain Controller.\
The **Netlogon service** uses that shared secret to create a **Secure Channel** between the computer and the Domain Controller.

Over this secure, encrypted channel, the server can safely send authentication data (like NTLM challenges/responses or Kerberos tickets) to the DC — without anyone being able to intercept or tamper with it.



