---
icon: person-biking-mountain
---

# Process Enumeration with NtAPI

Today we going to learn a new way to do process enum with NtAPI. we’ll be using `NtQuerySystemInformation`, which is a **syscall.** `NtQuerySystemInformation` is exported from the `ntdll.dll` module and therefore it will require the use of `GetModuleHandle` and `GetProcAddress`.

so we let’s start looking at the structure

```jsx
__kernel_entry NTSTATUS NtQuerySystemInformation(
  [in]            SYSTEM_INFORMATION_CLASS SystemInformationClass,
  [in, out]       PVOID                    SystemInformation,
  [in]            ULONG                    SystemInformationLength,
  [out, optional] PULONG                   ReturnLength
);
```

### Parameters

`[in] SystemInformationClass` :

Decides what type of system information the function returns.

Since the objective is process enumeration, the [SystemProcessInformation](https://learn.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntquerysysteminformation#systemprocessinformation) flag will be used.

**SystemProcessInformation**

Returns an array of **SYSTEM\_PROCESS\_INFORMATION** structures, one for each\
process running in the system.

These structures contain information about the resource usage of each process, including the number of threads and handles used by the process, the peak page-file usage, and the number of memory pages that the process has allocated.

> what is Peak page-file:-\
> Peak page-file usage in a Windows process is the maximum amount of virtual memory or paging file space (`pagefile.sys`) that the specific process has consumed at any single point during its entire lifetime

#### **SYSTEM\_PROCESS\_INFORMATION Structure**

The buffer pointed to by\
the _SystemInformation_ parameter should be large enough\
to hold an array that contains as many **SYSTEM\_PROCESS\_INFORMATION** and **SYSTEM\_THREAD\_INFORMATION** structures as\
there are processes and threads running in the system. This size is specified by the _ReturnLength_ parameter.

Each **SYSTEM\_PROCESS\_INFORMATION** structure has the following\
layout:

```jsx
typedef struct _SYSTEM_PROCESS_INFORMATION {
    ULONG NextEntryOffset;
    ULONG NumberOfThreads;
    BYTE Reserved1[48];
    UNICODE_STRING ImageName;
    KPRIORITY BasePriority;
    HANDLE UniqueProcessId;
    HANDLE InheritedFromUniqueProcessId;
    ULONG HandleCount;
    ULONG SessionId;
    PVOID Reserved3;
    SIZE_T PeakVirtualSize;
    SIZE_T VirtualSize;
    ULONG Reserved4;
    SIZE_T PeakWorkingSetSize;
    SIZE_T WorkingSetSize;
    PVOID Reserved5;
    SIZE_T QuotaPagedPoolUsage;
    PVOID Reserved6;
    SIZE_T QuotaNonPagedPoolUsage;
    SIZE_T PagefileUsage;
    SIZE_T PeakPagefileUsage;
    SIZE_T PrivatePageCount;
    LARGE_INTEGER Reserved7[6];
} SYSTEM_PROCESS_INFORMATION;
```

As of now we can see that it do give us lot of information however we’ll be focusing one the one which are meant to be mentioned

* `UNICODE_STRING ImageName` which contains the process name and `UniqueProcessId` which is the process ID. Additionally, `NextEntryOffset` will be used to move into the next element in the returned array.
* `NextEntryOffset` will be used to move into the next element in the returned array.

`[in, out] SystemInformation` :

A pointer to a buffer that will receive the requested information. The returned information will be in a form of a structure of type specified according to the `SystemInformationClass` parameter.

`[in] SystemInformationLength` :

The size of the buffer pointed to by the `SystemInformation` parameter, in bytes.

`[out, optional] ReturnLength` :

A pointer to a ULONG variable that will receive the actual size of the information written to `SystemInformation`.

### **Retrieve NtQuerySystemInformation's Address**

```jsx
// Function pointer
typedef NTSTATUS (NTAPI* fnNtQuerySystemInformation)(
	SYSTEM_INFORMATION_CLASS SystemInformationClass,
	PVOID                    SystemInformation,
	ULONG                    SystemInformationLength,
	PULONG                   ReturnLength
);

fnNtQuerySystemInformation pNtQuerySystemInformation = NULL;

// Getting NtQuerySystemInformation's address
pNtQuerySystemInformation = (fnNtQuerySystemInformation)GetProcAddress(GetModuleHandle(L"NTDLL.DLL"), "NtQuerySystemInformation");
if (pNtQuerySystemInformation == NULL) {
	printf("[!] GetProcAddress Failed With Error : %d\n", GetLastError());
	return FALSE;
}
```

**the typedef**

```c
typedef NTSTATUS (NTAPI* fnNtQuerySystemInformation)(    SYSTEM_INFORMATION_CLASS SystemInformationClass,    PVOID                    SystemInformation,    ULONG                    SystemInformationLength,    PULONG                   ReturnLength);
```

`typedef` just means "i'm creating a new type name". i’m creating a type called `fnNtQuerySystemInformation` that describes the shape of a function what arguments it takes and what it returns. think of it like a blueprint. `fn` prefix is just a naming convention maldev people say "this type represents a function pointer".

so now `fnNtQuerySystemInformation` is a type, just like `int` or `DWORD` is a type. except this type means "a pointer to a function that looks exactly like NtQuerySystemInformation".

***

**declaring the variable**

```c
fnNtQuerySystemInformation pNtQuerySystemInformation = NULL;
```

this is just declaring a variable using that type you just created. same as writing:

```c
int x = 0;
```

except instead of `int` the type is `fnNtQuerySystemInformation`. the variable `pNtQuerySystemInformation` is a function pointer a variable that will hold the memory address of a function. right now it's NULL because you haven't pointed it at anything yet.

***

**the cast**

```c
pNtQuerySystemInformation = (fnNtQuerySystemInformation)GetProcAddress(GetModuleHandle(L"NTDLL.DLL"), "NtQuerySystemInformation");
```

from inside out:

```c
GetModuleHandle(L"NTDLL.DLL")
```

gets the base address of ntdll where it's loaded in memory.

```c
GetProcAddress(modulehandle, "NtQuerySystemInformation")
```

walks ntdll's export table, finds `NtQuerySystemInformation`, returns its address as a raw `void*` it doesn't know what type it is, just a number.

```c
(fnNtQuerySystemInformation) ...
```

before this leme explain what casting is

### What casting is

you know how memory is just bytes right. like at address `0x1000` there's just raw bytes sitting there right

```
0x1000: 04 00 00 00 FF FF ...
```

those bytes mean nothing on their own. the meaning comes from HOW YOU READ THEM.

casting is just telling the compiler how to read those bytes.

```c
int* x = 0x1000       // read 4 bytes at 0x1000 as a single integer
char* y = 0x1000      // read 1 byte at 0x1000 as a character
PSYSTEM_PROCESS_INFORMATION z = 0x1000   // read bytes at 0x1000 as this struct
```

same address, same bytes, completely different meaning depending on the type.

so when you do:

```c
SystemProcInfo = (PSYSTEM_PROCESS_INFORMATION) HeapAlloc(...)
```

`HeapAlloc` just gave you an address, say `0x5000`. that's it. just a number.

the cast `(PSYSTEM_PROCESS_INFORMATION)` tells the compiler:

> when i access bytes at `0x5000`, read them as a `SYSTEM_PROCESS_INFORMATION` struct

so now when you write `SystemProcInfo->UniqueProcessId` the compiler knows:

> okay `UniqueProcessId` is at offset `+0x50` inside that struct, so go to `0x5000 + 0x50` and read it

without the cast the compiler has no idea what offsets to use because it doesn't know the struct layout. that's literally all casting is telling the compiler the shape of what's at that address.

this is the cast. `GetProcAddress` returns a generic `void*` it has no idea this address points to a function that takes those specific arguments. the cast tells the compiler "this address points to a function shaped exactly like `fnNtQuerySystemInformation`". now you can call it properly.

```c
pNtQuerySystemInformation = ...
```

stores that address into your variable.

```c
BOOL GetRemoteProcessHandle(LPCWSTR szProcName, DWORD* pdwPid, HANDLE* phProcess) {

	fnNtQuerySystemInformation   pNtQuerySystemInformation = NULL;
	ULONG                        uReturnLen1               = NULL,
                                 uReturnLen2               = NULL;
    PSYSTEM_PROCESS_INFORMATION  SystemProcInfo            = NULL;
    NTSTATUS                     STATUS                    = NULL;
	PVOID                        pValueToFree              = NULL;
	
	pNtQuerySystemInformation = (fnNtQuerySystemInformation)GetProcAddress(GetModuleHandle(L"NTDLL.DLL"), "NtQuerySystemInformation");
	if (pNtQuerySystemInformation == NULL) {
		printf("[!] GetProcAddress Failed With Error : %d\n", GetLastError());
		return FALSE;
	}

	pNtQuerySystemInformation(SystemProcessInformation, NULL, NULL, &uReturnLen1);

	SystemProcInfo = (PSYSTEM_PROCESS_INFORMATION)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, (SIZE_T)uReturnLen1);
	if (SystemProcInfo == NULL) {
		printf("[!] HeapAlloc Failed With Error : %d\n", GetLastError());
		return FALSE;
	}

	// Since we will modify 'SystemProcInfo', we will save its initial value before the while loop to free it later
	pValueToFree = SystemProcInfo;

	STATUS = pNtQuerySystemInformation(SystemProcessInformation, SystemProcInfo, uReturnLen1, &uReturnLen2);
	if (STATUS != 0x0) {
		printf("[!] NtQuerySystemInformation Failed With Error : 0x%0.8X \n", STATUS);
		return FALSE;
	}

	while (TRUE) {

		// Check the process's name size
		// Comparing the enumerated process name to the intended target process
		if (SystemProcInfo->ImageName.Length && wcscmp(SystemProcInfo->ImageName.Buffer, szProcName) == 0) {
			
			// Opening a handle to the target process, saving it, and then breaking 
			*pdwPid		= (DWORD)SystemProcInfo->UniqueProcessId;
			*phProcess	= OpenProcess(PROCESS_ALL_ACCESS, FALSE, (DWORD)SystemProcInfo->UniqueProcessId);
			break;
		}

		// If NextEntryOffset is 0, we reached the end of the array
		if (!SystemProcInfo->NextEntryOffset)
			break;

		// Move to the next element in the array
		SystemProcInfo = (PSYSTEM_PROCESS_INFORMATION)((ULONG_PTR)SystemProcInfo + SystemProcInfo->NextEntryOffset);
	}

	// Free using the initial address
	HeapFree(GetProcessHeap(), 0, pValueToFree);

	// Check if we successfully got the target process handle
	if (*pdwPid == NULL || *phProcess == NULL)
		return FALSE;
	else
		return TRUE;
}

```

**Function signature**

```c
BOOL GetRemoteProcessHandle(LPCWSTR szProcName, DWORD* pdwPid, HANDLE* phProcess)
```

***

**Variables**

```c
fnNtQuerySystemInformation   pNtQuerySystemInformation = NULL;
ULONG                        uReturnLen1               = NULL,
                             uReturnLen2               = NULL;
PSYSTEM_PROCESS_INFORMATION  SystemProcInfo            = NULL;
NTSTATUS                     STATUS                    = NULL;
PVOID                        pValueToFree              = NULL;
```

***

**Getting NtQuerySystemInformation**

```c
pNtQuerySystemInformation = (fnNtQuerySystemInformation)GetProcAddress(GetModuleHandle(L"NTDLL.DLL"), "NtQuerySystemInformation");
if (pNtQuerySystemInformation == NULL) {
    printf("[!] GetProcAddress Failed With Error : %d\n", GetLastError());
    return FALSE;
}
```

already covered this gets the address of `NtQuerySystemInformation` from ntdll at runtime, casts it to the function pointer type so you can call it. if it fails something is seriously broken.

***

**Two call pattern**

```c
pNtQuerySystemInformation(SystemProcessInformation, NULL, NULL, &uReturnLen1);
```

first call with NULL buffer kernel returns `STATUS_INFO_LENGTH_MISMATCH` but writes the required size into `uReturnLen1`.

```c
SystemProcInfo = (PSYSTEM_PROCESS_INFORMATION)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, (SIZE_T)uReturnLen1);
if (SystemProcInfo == NULL) {
    printf("[!] HeapAlloc Failed With Error : %d\n", GetLastError());
    return FALSE;
}
```

allocate exactly the right amount of heap memory, cast it to `PSYSTEM_PROCESS_INFORMATION` so the compiler knows the shape of what's in there.

***

**Saving the original pointer**

```c
pValueToFree = SystemProcInfo;
```

we are about to walk through the buffer by moving `SystemProcInfo` forward in memory. by the end of the loop `SystemProcInfo` will be pointing somewhere in the middle of the heap buffer, not the start. you cannot `HeapFree` from the middle you must free from the original address. so you save it here before touching it.

***

**Second call filling the buffer**

```c
STATUS = pNtQuerySystemInformation(SystemProcessInformation, SystemProcInfo, uReturnLen1, &uReturnLen2);
if (STATUS != 0x0) {
    printf("[!] NtQuerySystemInformation Failed With Error : 0x%0.8X \n", STATUS);
    return FALSE;
}
```

now with a real buffer. kernel walks `PsActiveProcessHead`, writes all `SYSTEM_PROCESS_INFORMATION` structs back to back into your heap buffer. `STATUS 0x0` = success, anything else = failure.

***

**The while loop walking the buffer**

```c
while (TRUE) {
    if (SystemProcInfo->ImageName.Length && wcscmp(SystemProcInfo->ImageName.Buffer, szProcName) == 0) {
        *pdwPid    = (DWORD)SystemProcInfo->UniqueProcessId;
        *phProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, (DWORD)SystemProcInfo->UniqueProcessId);
        break;
    }
```

`ImageName` is a `UNICODE_STRING` struct it has a `Length` field and a `Buffer` field. checking `ImageName.Length` first makes sure the name is not empty before comparing some system processes have no name. `wcscmp` compares the process name against your target. on match, write the PID and open the handle, then break.

```c
    if (!SystemProcInfo->NextEntryOffset)
        break;
```

`NextEntryOffset` tells you how many bytes to jump forward to reach the next `SYSTEM_PROCESS_INFORMATION` entry. when it is 0 you have hit the last process in the buffer nothing more to walk, break out.

```c
    SystemProcInfo = (PSYSTEM_PROCESS_INFORMATION)((ULONG_PTR)SystemProcInfo + SystemProcInfo->NextEntryOffset);
}
```

this is the pointer walking. `ULONG_PTR` is just an integer big enough to hold a pointer (8 bytes on x64). you cast `SystemProcInfo` to it so you can do raw byte arithmetic add `NextEntryOffset` bytes to move to the next struct. then cast back to `PSYSTEM_PROCESS_INFORMATION` so the compiler knows what fields to read next iteration. you cannot do math directly on a struct pointer like this without the `ULONG_PTR` cast.

***

**Cleanup and return**

```c
HeapFree(GetProcessHeap(), 0, pValueToFree);
```

free using the original address you saved before the loop. not `SystemProcInfo` because that is now pointing somewhere in the middle.

```c
if (*pdwPid == NULL || *phProcess == NULL)
    return FALSE;
else
    return TRUE;
```

if the loop finished without a match both outputs are still NULL return FALSE. if match was found and break fired both are populated return TRUE.
