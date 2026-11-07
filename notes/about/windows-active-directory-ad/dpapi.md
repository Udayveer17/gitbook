---
icon: shield-keyhole
---

# DPAPI

## Understanding DPAPI and How Attackers Abuse It

<figure><img src="../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

### What is DPAPI and Why Does it Even Exist

Before DPAPI, every developer who wanted to store a password securely had to implement their own encryption. Some did it badly, some did not bother at all. Microsoft got tired of this and baked encryption directly into Windows so any application could call two simple functions and get proper protection without knowing anything about crypto.

That system is DPAPI, the Data Protection API, and it has been part of Windows since Windows 2000. Chrome wants to save your Facebook password? It hands it to DPAPI, gets back an encrypted blob, and saves that. When it needs the password again, it hands the blob back and gets the plaintext. Chrome never touches a single key or algorithm. DPAPI handles everything.

The things DPAPI protects include saved RDP credentials, browser passwords, WiFi keys, Windows Vault entries, Outlook passwords, VPN credentials, and cookies. Basically anything sensitive that Windows or a third party app wants to store safely on disk.

The entire system is built on one idea. The encryption is tied to the user's Windows login password. Only the user who encrypted the data can decrypt it. Sounds solid until you understand how it works under the hood, because the moment an attacker gets that password, everything falls apart.

### The Two Keys You Need to Understand

There are two keys in DPAPI and people mix them up constantly. They are the MasterKey and the Session Key and they do completely different jobs.

#### The MasterKey

The MasterKey is not really a key in the traditional sense. It is 512 bits of completely random data that Windows generates for your user account. It never directly encrypts anything. Its only purpose is to produce the real encryption key when needed.

Because this secret is so sensitive, Windows cannot store it as plaintext on disk. So it locks it. It takes your login password, runs it through SHA1, feeds that hash into PBKDF2 along with a 16 byte random Salt1 and an iteration count, and uses the output to encrypt the 512 bit secret with 3DES or AES. That encrypted file sits on disk at:

C:\Users\you\AppData\Roaming\Microsoft\Protect\S-1-5-21-yourSID\\

The filename is a GUID. You can have multiple MasterKey files in that folder because they expire after three months and a new one gets created. Windows never deletes old ones because old blobs on disk still reference them.

The salt and iteration count inside the MasterKey file are plaintext. The only locked thing is the 512 bit secret itself, protected by your password.

#### The Session Key

The Session Key is the actual worker. It is the symmetric key that directly encrypts and decrypts your data. It gets derived fresh every single time DPAPI encrypts something, using the MasterKey secret plus a brand new 16 byte Salt2 fed into HMAC-SHA1. That output is the Session Key.

The Session Key is never stored. After encryption Windows wipes it from memory using SecureZeroMemory which zeroes those bytes so even a memory dump cannot recover it. The only way to get it back is to re-derive it, which needs the MasterKey, which needs your password.

### What is Inside the Encrypted Blob

The blob is what DPAPI returns to the application after encryption. Most of the fields inside it are readable plaintext. Only the actual payload at the bottom is encrypted. The blob stores the MasterKey GUID so DPAPI knows which file to load from disk later. It stores Salt2 so the Session Key can be re-derived. It stores the algorithm used like AES256 or 3DES. It stores an HMAC which is an integrity fingerprint proving nobody tampered with the blob. And at the end it stores your actual encrypted data.

The salt being plaintext is not a flaw. It was never meant to be secret. The entire security of DPAPI rests on the password being secret, nothing else.

### How the Encryption Happens

When Chrome calls CryptProtectData, the function lives in crypt32.dll and does zero crypto itself. It just packages your data and sends it to LSASS over a local named pipe called NCALRPC which never leaves the machine.

LSASS is the privileged Windows process that holds your cached login credentials in memory since you logged in. It figures out who is calling by reading the SID from the process token, goes to disk and finds the current MasterKey file, reads Salt1 from it in plaintext, uses the cached password to run PBKDF2 and unlock the 512 bit secret, generates fresh Salt2, runs HMAC-SHA1 to produce the Session Key, encrypts your data with AES256, builds the blob, sends it back to crypt32.dll, and then immediately wipes the Session Key from memory.

The application saves the blob to disk. Done.

### How an Attacker Gets the Plaintext Credentials

The attacker wants the plaintext creds sitting in encrypted files at:

C:\Users\victim\AppData\Roaming\Microsoft\Credentials\\

Their first move is to parse one of those blob files and read the plaintext headers. This gives them the MasterKey GUID which tells them exactly which MasterKey file to grab from the Protect folder. They also get Salt2 which they save for later. They copy both files offline to their own machine and do everything from there. No interaction with the live system means nothing gets logged.

Getting Salt1 is trivial. They just open the MasterKey file and read it. It is sitting there in plaintext. Same with the iteration count. The only locked thing is the encrypted 512 bit secret. They run:

```
dpapi.py masterkey -file {GUID} -sid S-1-5-21-xxx -password 'Password123'
```

The password goes through SHA1, then PBKDF2 with Salt1 and the iteration count, and the output decrypts the 512 bit MasterKey secret. Now they have it.

They feed that MasterKey secret and Salt2 into HMAC-SHA1 to re-derive the Session Key and run:

```
dpapi.py credential -key 0xd9a570722f... -file 4D435053
```

The HMAC integrity check passes, AES256 decrypts the payload, and the username and password print out in plaintext.

### What if There is No Password

If the target user is currently logged in, the MasterKey is already decrypted in LSASS memory. Mimikatz can pull it directly with sekurlsa::dpapi, no password needed.

If the attacker has domain access they can ask the Domain Controller to decrypt the MasterKey using dpapi::masterkey /rpc in Mimikatz. Every domain joined machine sends a backup of each MasterKey to the DC encrypted with a domain RSA public key. The DC holds the private key and can decrypt any MasterKey for any user in the domain.

And if the attacker gets to the DC itself they can steal that RSA private key entirely with lsadump::backupkeys. One key, every user's secrets, forever. This is why that domain backup key is called the crown jewel of Active Directory.

### The Bottom Line

DPAPI is well designed. The layered approach means full disk access alone does not immediately hand over everything. The attacker still needs the password.

But the moment that password is compromised the entire chain unravels. Every saved credential on that machine becomes readable. And in a domain environment with the backup key stolen, the blast radius covers every single user in the domain. The whole system is only as strong as the user's password. That is both the elegance and the weakness of DPAPI.
