---
icon: memory
---

# LSA & LSASS

## LSA, LSASS & Credential Dumping

_how Windows handles your creds under the hood, and why attackers are obsessed with it_

***

### Before We Start

fr every single article about credential dumping just goes straight to "run mimikatz get passwords" with zero context. we're not doing that. we're going from the very beginning, what LSA even is, what happens when you type your password, all the way to how procdump and pypykatz actually work. no skipping.

***

### What Even Is LSA?

normal example first. imagine a corporate office building with a security guard at the front desk. you walk in, he checks your ID, checks if you're on the access list, then hands you a visitor badge for the rest of the day.

that guard is **LSA, Local Security Authority.**

LSA is not a file or a program. it's a **role**, basically a rulebook Windows gave to one specific part of the OS. its only job is answering one question on repeat: _who are you, are you allowed in, and what can you touch?_

that's it. it doesn't run apps, doesn't store files, doesn't do networking. just decides yes or no. people call it "just a concept" which confuses everyone fr, what they mean is LSA is a responsibility, not one single thing you can point to. the actual process carrying out that responsibility is LSASS.

***

### Then What Is LSASS?

if LSA is the rulebook, `lsass.exe` is the worker actually reading and enforcing it.

it's a Windows process, you can see it in Task Manager right now. it runs 24/7 and handles every single authentication request on the machine. LSA is the job description, LSASS is the employee doing the job.

try killing `lsass.exe` on a live system. Windows BSODs instantly. every login, every permission check, every auth request goes through this one process. it's not optional at all.

***

### What Happens at the Login Screen

when you press `Ctrl+Alt+Delete`, that's not just a shortcut. it's a hardware-level interrupt called the Secure Attention Sequence. only `Winlogon.exe` is allowed to catch it, nothing else, no app, no malware. this exists so nobody can fake the login screen on you.

Winlogon wakes up, launches `LogonUI.exe` which is the actual screen you see, you type your password, LogonUI hands it to Winlogon, Winlogon passes it to LSASS through a DLL called `secur32.dll`. that DLL is just the bridge between the two, nothing fancy.

now here's where people always skip the good part. LSASS doesn't authenticate you itself. it has **authentication packages** loaded inside its own process space, basically DLLs running inside LSASS, and each one handles a different protocol. `msv1_0.dll` handles NTLM for local accounts, `kerberos.dll` handles Kerberos for domain accounts, `wdigest.dll` handles legacy WDigest. and then there's `lsasrv.dll` which is the negotiator, it decides which package gets your credential.

***

### How lsasrv.dll Picks: Domain or Local?

`lsasrv.dll` has a Negotiate function. when your credential comes in it asks one thing, does this username have a domain prefix or not?

you typed `CORP\Administrator`? lsasrv sees the domain, hands it to `kerberos.dll`. Kerberos sends an AS-REQ to the Domain Controller, DC checks Active Directory, sends back a TGT. you're in as a domain user.

you typed just `Administrator`? lsasrv treats it as local, hands it to `msv1_0.dll`. that DLL hashes your password with MD4 to get the NT Hash, then checks the SAM database on this machine to compare. match means success.

two completely different code paths, two completely different storage locations. that's why domain and local accounts are separate attack surfaces, they literally go through different systems.

#### What Is SAM?

SAM, Security Accounts Manager, is just a local database. it stores one thing: username mapped to NT Hash. your password is never stored in plaintext ever. Windows hashes it with MD4 and only keeps the result. at login it hashes what you typed and compares. match means you're in.

SAM lives at `C:\Windows\System32\config\SAM` but you can never copy it while Windows is running, SYSTEM has it permanently locked. it's also encrypted on disk with the syskey which we'll get to next.

***

### The Access Token

once LSASS verifies you it creates an **Access Token**, a kernel object that follows your entire session everywhere.

back to the office analogy. after the guard checks your ID he gives you a visitor badge. nobody stops you at every door asking for ID again, they just check the badge. that badge is your token.

it holds your SID, every group SID you belong to, your privileges like `SeDebugPrivilege` and `SeImpersonatePrivilege`, and your integrity level. every process you launch gets a copy of this token. CMD gets it, your scripts get it, everything. LSA made one decision at login and it travels with you for the whole session.

this is also why token manipulation is so powerful in post-exploitation. steal or impersonate someone's token and Windows thinks you're them, fr.

***

### Syskey and the Registry

Windows has a master key called the **Syskey**, also called the bootkey. generated at every boot, and the sneaky part is it's not stored in one place. it's split across four registry keys under `HKLM\SYSTEM\CurrentControlSet\Control\Lsa`, the subkeys JD, Skew1, GBG, and Data. none of those four alone is the syskey, you need all four plus a specific algorithm. Windows does this automatically at boot.

syskey encrypts the SAM database on disk, encrypts LSA Secrets in the registry, and at boot it derives two in-memory keys inside LSASS: `h3DesKey` and `hAesKey`. think of syskey as the parent and those two as its kids. the kids do the actual work inside LSASS memory, the parent stays in the registry and never touches LSASS directly.

`h3DesKey` and `hAesKey` are what LSASS uses to encrypt stuff while it sits in memory. WDigest plaintext gets encrypted with `h3DesKey` before being stored. Kerberos keys and tickets get encrypted with `hAesKey`. NT hashes do not get encrypted, more on why in a sec.

#### LSA Secrets

LSASS memory is volatile, reboot and it's gone. but some creds need to survive reboots. those live in the registry as **LSA Secrets** at `HKLM\SECURITY\Policy\Secrets`, encrypted with syskey. things like service account passwords, the machine's AD account password, scheduled task creds, IIS app pool passwords, and DCC2 hashes.

completely separate attack surface from LSASS memory. different location, different tools, different data.

***

### ProcDump

ProcDump is a legit Microsoft Sysinternals tool, a debugging utility Microsoft themselves ship. it's Microsoft-signed which is a big reason red teamers use it. under the hood it calls one API: `MiniDumpWriteDump`. that API tells Windows to take everything inside `lsass.exe`'s memory and write it to a file on disk. Windows just does it, no filtering, no parsing, raw snapshot.

```bash
procdump.exe -ma lsass.exe lsass.dmp
```

`-ma` means full memory dump, you want everything. what you get on disk is a `.dmp` file in Microsoft's MINIDUMP format. not encrypted. raw copy of LSASS memory. so it has the NT hashes from `msv1_0.dll`, the AES-encrypted WDigest plaintext from `wdigest.dll`, the encrypted Kerberos stuff from `kerberos.dll`, and critically the `h3DesKey` and `hAesKey` from `lsasrv.dll`.

the decryption keys are in the dump right next to the encrypted data. the dump decrypts itself. you don't need the syskey separately, the derived keys are already in there.

the smart part operationally is that the noisy bit, touching LSASS, happens fast on target. all the slow parsing happens offline on your own machine where no EDR is watching. downside is the file is big, usually 50 to 150MB, and any decent EDR watches specifically for `MiniDumpWriteDump` being called against `lsass.exe`.

***

### pypykatz

pypykatz is a pure Python rewrite of Mimikatz's parsing logic. you don't need a Windows machine, just Python anywhere.

```bash
pip install pypykatz
pypykatz lsa minidump lsass.dmp
```

four things happen under the hood when you run that.

first it parses the MINIDUMP format. the `.dmp` has a header describing what memory regions were captured and where they sit in the file. pypykatz reads it to rebuild lsass's virtual address space so it can navigate the file like real memory.

second it finds where each DLL was loaded. it scans for the module list to get base memory addresses of `msv1_0.dll`, `kerberos.dll`, `wdigest.dll`, and `lsasrv.dll`. all the credential structures are at offsets from those base addresses.

third, and most important, it finds the session keys. it looks inside `lsasrv.dll`'s memory region, searches for a known byte signature that varies per Windows version, and locates `h3DesKey` and `hAesKey`. once it has those it can decrypt everything.

fourth it walks each auth package. for NT hashes it finds the `MSV1_0_LIST` linked list, walks every node (each node is one logon session), pulls username, domain, NT hash, LM hash directly. no decryption needed, hashes are stored as-is. for WDigest it finds `l_LogSessList`, each entry has a 3DES-encrypted password field, decrypts it with `h3DesKey`, plaintext comes out. for Kerberos it finds `KerbLogonSessionList`, extracts TGTs and service tickets, decrypts with `hAesKey`, also grabs the AES256, AES128, DES long-term keys.

```bash
# basic parse
pypykatz lsa minidump lsass.dmp

# json output, good for piping
pypykatz lsa minidump lsass.dmp -o json

# save to file
pypykatz lsa minidump lsass.dmp -o json > creds.json

# SAM + SYSTEM hive path, no lsass dump needed
pypykatz registry --sam sam.hiv system.hiv
```

that last one is the SAM attack path. grab the two hives with `reg save HKLM\SAM sam.hiv` and `reg save HKLM\SYSTEM system.hiv`, or pull them from a Volume Shadow Copy. pypykatz gets the syskey from SYSTEM, decrypts SAM, gives you NT hashes for every local account. no LSASS needed at all.

***

### Quick Reference

| Attack Path       | What You Get                                         | Commands                                              |
| ----------------- | ---------------------------------------------------- | ----------------------------------------------------- |
| LSASS dump        | NT hashes, plaintext if WDigest on, Kerberos tickets | `procdump -ma lsass.exe` then `pypykatz lsa minidump` |
| SAM + SYSTEM hive | Local account NT hashes only                         | `reg save` then `pypykatz registry`                   |
| LSA Secrets       | Service passwords, machine account hash, DCC2 hashes | `mimikatz lsadump::secrets`                           |
| Kerberos tickets  | TGTs for Pass-the-Ticket                             | `mimikatz sekurlsa::tickets`                          |
