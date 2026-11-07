---
icon: hexagon-vertical-nft
---

# UAC

## What is UAC (User Account Control)?

ok so you know that popup Windows throws at you when you try to run something and it's like "do you want to allow this app to make changes to your device?" yeah that's UAC fr. most people just click yes without thinking but there's actually a lot going on under the hood and if you're into security you need to understand this properly.

let me break it down.

***

### The Core Idea

UAC is basically Windows saying "nah not everything gets to run as admin just because you're an admin." every process runs at the privilege level of whoever launched it, nothing more nothing less.

but here's the thing, UAC doesn't do this alone. at its core it uses something called **MIC (Mandatory Integrity Control)** which is the real mechanic behind all of this. MIC assigns an **Integrity Level (IL)** to every user, every process, and every resource on the system.

the rule is dead simple:

> a process can only access resources at the same or lower IL than itself.

so even if the normal Windows permission system (DACL) says you have access to something, MIC can still block you if your IL isn't high enough. MIC wins, no cap.

***

### Integrity Levels

there are 6 integrity levels in Windows, going from lowest to highest:

| Level         | Who Uses It                                                        |
| ------------- | ------------------------------------------------------------------ |
| **Untrusted** | anonymous processes, no login context at all                       |
| **Low**       | sandboxed stuff like Chrome renderer, Internet Explorer            |
| **Medium**    | standard users and the admin filtered token (most things run here) |
| **High**      | admin elevated token, only after UAC is accepted                   |
| **System**    | Windows kernel, core services like lsass.exe                       |
| **Installer** | MSI trusted class, can uninstall anything below it                 |

the jump that matters the most for us in security is **Medium to High**. that's literally the gap every UAC bypass technique is trying to cross.

***

### The Two Tokens Thing (this is actually so important)

ok so this is where a lot of people get confused. when an admin logs into Windows they don't just get one token, they get two at the same time:

* **Filtered Token** at Medium IL - used for literally everything by default
* **Elevated Token** at High IL - only handed out when the UAC prompt is accepted

so even if you're a local admin, your shell is starting at Medium IL. this is why on a pentest you can get a shell as a local admin and still can't dump SAM or touch protected paths. you're sitting on that filtered token and it's cooked until you elevate.

standard users only get one token at Medium IL, no elevated option at all.

***

### UAC Notification Levels

UAC isn't always the same, it has 4 different settings and depending on the machine you're on it could be any of them:

**Always Notify** - prompts for literally everything, even Windows setting changes. most strict, most annoying.

**Notify only when programs try to make changes** - this is the default. prompts when apps try to install or modify things but not for Windows setting changes made by admins.

**Notify only when programs try to make changes (no secure desktop)** - same as above but the prompt doesn't run on a secure desktop. slightly weaker and we'll explain why in a sec.

**Never Notify** - UAC is basically off at this point. admins always run with their elevated token, no questions asked.

***

### How UAC Actually Works Internally

aight this is where it gets actually interesting. when you right click something and hit "Run as Administrator" here's what's actually happening under the hood step by step:

#### Step 1: ShellExecute with runas verb

Windows makes a `ShellExecute` API call with the `runas` verb. this is basically Windows saying "yo someone wants elevation."

#### Step 2: AppInfo picks it up

the request gets forwarded to the **Application Information Service (AppInfo)**, which runs as `appinfo.dll` inside `svchost.exe`. AppInfo is the gatekeeper fr. if this service is stopped, nothing on the entire system can elevate. every admin request would just fail silently, which is kinda wild.

#### Step 3: AppInfo reads the manifest file

every executable has an embedded **manifest file** baked into it that tells Windows how it wants to run. AppInfo reads this before doing anything else. inside the manifest there's a field called `requestedExecutionLevel` and it can be one of three values:

**`asInvoker`** - just run at whatever level the parent process is at. no prompt, no drama. this is what most normal apps use.

**`requireAdministrator`** - this app always needs admin, no exceptions. UAC prompt will always appear. think installers, regedit, stuff like that.

**`highestAvailable`** - this one is the smart one. it checks who's running it first. if a standard user runs it, it gets Medium IL with no prompt. if an admin runs it, UAC prompt appears and it gets High IL. Task Manager uses this which is why it behaves differently depending on who opens it.

#### Step 4: consent.exe on the Secure Desktop

if elevation is needed, AppInfo fires up **consent.exe** which is what actually shows you the UAC prompt. but here's the thing, it doesn't run it on your normal desktop. it runs it on a **Secure Desktop**.

the secure desktop is a completely separate desktop session, isolated from everything you have open. the reason this matters is that on a normal desktop, other processes can literally send fake mouse clicks and keyboard inputs to UI elements. so without the secure desktop, malware could just silently click "Yes" on the UAC prompt without you ever knowing. the secure desktop shuts that down completely because only consent.exe is running there, nothing else can touch it.

#### Step 5: Elevated token handed back

if you click Yes, AppInfo grabs the user's Elevated Token and uses it to launch the process with High IL. it also re-parents the new process so its parent PID points back to the shell you launched it from, not AppInfo itself. sneaky but makes sense.

the full flow goes like this:

1. you click "Run as Admin"
2. `ShellExecute` fires with the `runas` verb
3. AppInfo picks it up and reads the manifest
4. consent.exe shows the prompt on the secure desktop
5. you click Yes and the process launches with a High IL elevated token

***

### Auto-Elevation (The Exception)

ok so here's where it gets actually spicy. some Microsoft binaries straight up skip the UAC prompt entirely. they self-elevate silently without ever touching consent.exe. this is called **auto-elevation** and this is literally the attack surface for most UAC bypass techniques.

for a binary to auto-elevate, all three of these have to be true at the same time:

* digitally signed by Microsoft
* located in a trusted directory like `C:\Windows\System32\`
* `autoElevate=true` in the manifest

if all three pass, AppInfo just skips consent.exe and silently hands the binary a High IL token. no prompt, no user interaction, nothing.

examples of binaries that do this: `fodhelper.exe`, `eventvwr.exe`, `sdclt.exe`.

this is exactly what UAC bypass techniques abuse. the idea is always the same, find something the auto-elevating binary does before it actually elevates (like reading a registry key or loading a COM object) and hijack that thing. your payload rides up to High IL along with it. clean.

***

### References

* [Microsoft Docs: How UAC Works](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/user-account-control/how-it-works)
* [HackTricks: UAC and Authentication](https://book.hacktricks.wiki/en/windows-hardening/authentication-credentials-uac-and-efs/uac-user-account-control.html#policies-for-installing-software-on-windows)
* [HackTricks: Integrity Levels](https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/integrity-levels.html#integrity-levels-1)
* [Fundamentals of UAC Part 01 by Sachin Sir](https://medium.com/@SachinSir/fundamentals-of-uac-user-account-control-part-01-d98d423c7289)
* [Fundamentals of UAC Part 02 by Sachin Sir](https://medium.com/@SachinSir/fundamentals-of-uac-user-account-control-part-02-7085ca27f465)
