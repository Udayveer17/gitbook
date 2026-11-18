---
icon: circle-up-left
---

# Return Address Spoofing

> This article is heavily influenced by https://hulkops.gitbook.io/blog/red-team/x64-return-address-spoofing

## Return Address Spoofing

AntiVirus or EDR solutions determine if an activity is malicious based on behavioural thresholds. If a process's activity crosses this threshold, it gets categorized as suspicious or outright malicious and is probably terminated. The threshold, however, is different for all processes. Trusted processes, like system services, have a higher threshold as opposed to the process of a never-seen or unsigned executable. There are various activities that are monitored to identify suspicious activities. Typically, one such activity is the execution of suspicious APIs from unbacked memory regions. For instance, if a shellcode that has been loaded into the memory starts executing WinAPIs like "InternetConnectA", "HttpOpenRequestA", or "HttpSendRequestA", the process can be categorized as malicious with certainty.

> **Unbacked memory regions** are allocated areas of a process's virtual memory that are **not mapped to a physical file** on the disk. Unlike **backed memory** (type `IMG`), which corresponds to executable images or loaded modules like DLLs, unbacked memory (type `PRV` or `Private`) exists only in RAM.
>
> These regions are frequently used by **malware** and **process injection** techniques (such as reflective DLL injection or shellcode execution) to store payloads without dropping files to the hard drive. Because they often possess **Read, Write, and Execute (RWX)** permissions, the presence of executable code in unbacked memory is a primary **Indicator of Compromise (IOC)** for security tools and EDRs.

To determine the memory location from which a WinAPI is called, one just needs to look at its return address. This is the address from which the program will continue execution after returning from the WinAPI. By modifying the return address, we can make it look like the function is being called from a different location. This technique is called "**Return Address Spoofing**". While this technique is not new or perfect and can be detected as it breaks the call stack chain, it is a fundamental technique to build upon. There are various blogs and great resources on this technique, and going through them inspired me to develop a POC that works with all WinAPIs.

Before we get into the implementation, there are two things worth going over - how the stack works and the x64 calling convention. Both of these are foundational to understanding what we're actually doing when we spoof a return address.

### Stack Basics

A stack is a data structure which follows LIFO = Last In First Out. This means that the last element added to the stack is the first one to be removed and if we push too many elements it says stack overflow. The terms Execution Stack, Program Stack, and Hardware Stack are all used simultaneously - they all refer to the same thing.

In programming terms, putting an item on top of the stack is called **push** and removing an item is called **pop**.

<figure><img src="../.gitbook/assets/image (91).png" alt=""><figcaption></figcaption></figure>

There are some basic operations that allow us to perform different actions on a stack.

* **Push**: Add an element to the top of a stack
* **Pop**: Remove an element from the top of a stack
* **IsEmpty**: Check if the stack is empty
* **IsFull**: Check if the stack is full
* **Peek**: Get the value of the top element without removing it

### x64 Calling Convention

When one function calls another on 64-bit Windows, they need an agreement on three things: where arguments go, where the return value comes back, and who owns which piece of the stack. That agreement is the calling convention - Microsoft x64, also known as "fastcall".

#### Passing arguments

The first four arguments go in registers, in fixed positions:

| Arg # | Integer/pointer | Float/double |
| ----- | --------------- | ------------ |
| 1     | RCX             | XMM0         |
| 2     | RDX             | XMM1         |
| 3     | R8              | XMM2         |
| 4     | R9              | XMM3         |

Anything beyond four goes on the stack. Note the position rule: the Nth argument uses the Nth slot, whether that's the integer or the XMM column. So `f(int a, double b, int c)` uses RCX, XMM1, R8. The real split here is integer/pointer types vs floating point, not scalar vs non-scalar.

#### Shadow space

Before the `call`, the caller reserves 32 bytes (4 x 8) on the stack, **always**, even if the function takes zero arguments. This is the shadow (or home) space. It's scratch space owned by the callee. The callee _may_ use it to spill RCX/RDX/R8/R9, which is handy for debuggers, for functions that need those registers for something else, or for varargs functions.

#### The stack frame layout

RSP points at the lowest address of the current frame, and the stack grows downward. When `call` executes, the CPU pushes the return address, so right after entering the callee:

```
[rsp+30h]  arg6
[rsp+28h]  arg5
[rsp+20h]  shadow for R9
[rsp+18h]  shadow for R8
[rsp+10h]  shadow for RDX
[rsp+08h]  shadow for RCX
[rsp+00h]  return address   <- RSP
```

Reading bottom to top: the return address is the first thing at `[rsp]`, the 32-byte shadow space follows, and then stack arguments (5th, 6th, ...) sit above that. Everything above `[rsp]` here was set up by the **caller**; the callee's own locals and pushes go _below_ RSP (at lower addresses).

#### Return values

* Integers and pointers: **RAX**
* Floats and doubles: **XMM0**

#### Putting it together

Calling `foo(1, 2, 3, 4, 5, 6)` from the caller's side looks like this. The offsets here are measured before `call` pushes the return address, so the stack args sit right above the shadow space at `[rsp+20h]` and `[rsp+28h]`. Once `call` executes and pushes the return address, RSP drops by 8 and those same slots shift to `[rsp+28h]` and `[rsp+30h]` from the callee's perspective.

```nasm
sub  rsp, 38h
mov  qword [rsp+28h], 6    ; arg6
mov  qword [rsp+20h], 5    ; arg5
mov  ecx, 1
mov  edx, 2
mov  r8d, 3
mov  r9d, 4
call foo
add  rsp, 38h
```

The `38h` comes from 20h shadow + 10h stack args + 8 to keep RSP 16-byte aligned at the `call`, since the return address push is 8 bytes and the stack must be 16-byte aligned at function entry minus that push.

#### Register preservation

* **Volatile** (callee can trash): RAX, RCX, RDX, R8-R11, XMM0-XMM5
* **Non-volatile** (callee must restore): RBX, RBP, RDI, RSI, RSP, R12-R15, XMM6-XMM15

For maldev, this is the foundation of a lot of things: writing shellcode that calls WinAPI functions, building syscall stubs, crafting ROP chains, and writing BOFs or inline assembly. Get the shadow space or alignment wrong and you'll get crashes, typically inside functions using `movaps` on a misaligned stack.

### Implementation

Now that the calling convention is clear, let's look at what we're actually doing. Take this simple code:

```c
#include <Windows.h>
#include <stdio.h>

int main() {
	MessageBoxA(NULL, "TEST", "TEST", MB_OK);
}
```

Here's how it looks inside a debugger:

<figure><img src="../.gitbook/assets/image (90).png" alt=""><figcaption></figcaption></figure>

[image.png](https://claude.ai/chat/image%201.png)

The breakpoint is set at the first instruction of **MessageBoxA**. Since this API is called within the "main()" function, we can see its return address at the top of the stack. Our objective will be to modify this value to an address within a trusted DLL, such as KERNEL32.DLL. This will make it look as if MessageBoxA is being called from KERNEL32.DLL. When the MessageBoxA API returns, the execution will be directed at the address in KERNEL32.DLL, and the instructions at this address will be executed.

The return address needs to be selected in such a way that the corresponding instructions redirect execution flow back to us, or else the program will crash. Such instructions are usually referred to as gadgets. We will use the following gadget in our program `jmp QWORD PTR [rbx]`.

The address of our liking is selected and its pointer is placed into the "rbx" register. When the gadget in KERNEL32.DLL gets executed, the control flow will be directed back to the address stored in "rbx", giving us the control.

#### Finding the gadget

The following function retrieves the memory location of our gadget from a loaded DLL.

```c
UINT64 FindGadget() {

	PBYTE hModule = GetModuleHandleA("kernel32");
	DWORD dwSize = ((PIMAGE_NT_HEADERS64)(hModule + ((PIMAGE_DOS_HEADER)hModule)->e_lfanew))->OptionalHeader.SizeOfImage;
	UINT64 pGadget = NULL;

	// Searching for the Bytes 0xFF 0x23, which corresponds to the instruction "jmp QWORD PTR [rbx]"
	for (int i = 0; i < dwSize - 1; i++) {

		if (hModule[i] == 0xff && hModule[i + 1] == 0x23) {
			pGadget = hModule + i;
			break;
		}

	}

	return pGadget;
}
```

but before we move on there is a catch in this. The issue in this code is we are **scanning the entire image** (all sections: `.text`, `.rdata`, `.data`, etc.). The bytes `FF 23` can appear in **data** (strings, constants, vtables) and won't be a valid instruction. We should only scan **executable sections** (`IMAGE_SCN_MEM_EXECUTE`). **No instruction-boundary check.** Even within `.text`, the `FF` byte might be part of a longer instruction (e.g., a displacement or immediate of a preceding instruction), making it a false positive.

example:

```nasm
mov rax, 0x2300FF0000000001   ; this encodes as: 48 B8 01 00 00 00 00 FF 23 00
```

Bytes `FF 23` appear inside this instruction as an **immediate value**, not as a standalone `jmp [rbx]`. If my scanner hits those bytes and returns that address, we jump into the **middle** of an instruction, CPU decodes garbage, crash. The best fix is using a proper disassembler like capstone that walks the instruction boundaries correctly instead of doing raw bytes scanning. For now I have just read about it that the fix for this code will be capstone but I have no idea how we'll do it, anyways we'll look into it later on.

> we'll be working on this later once we understand the full code of this nigga

#### Setting up the config

We will store all the required arguments inside a structure which is defined as follows:

```c
typedef struct _STACK_CONFIG {

	PVOID pRopGadget;
	PVOID pTarget;
	DWORD dwNumberOfArgs;
	PVOID pEbx;
	PVOID pArgs;

}STACK_CONFIG, * PSTACK_CONFIG;
```

* "pRopGadget" will hold the address of our gadget
* "pTarget" will contain the address of the function which needs to be called. e.g. MessageBoxA, VirtualAlloc, etc.
* "dwNumberOfArgs" to specify the number of arguments our target function needs. e.g. 4 for MessageBoxA
* "pEbx" to hold the address of our liking when the gadget is executed
* "pArgs" pointer to the arguments of our target function

We will use a function to initialize this structure and the required arguments for our target function.

```c
BOOL SetupConfig(PVOID pGadget, PSTACK_CONFIG pConfig, PVOID pTarget, DWORD dwArgCount, ...) {

	va_list arg_list;

	// Initializing the Struct values

	// To Keep The Stack Aligned, The Number Of Arguments Are Modified 
	pConfig->dwNumberOfArgs = (dwArgCount > 4) ? dwArgCount : 4;
	pConfig->dwNumberOfArgs += (dwArgCount % 2 != 0) ? 1 : 0;
	pConfig->pTarget = pTarget;
	pConfig->pRopGadget = pGadget;
	pConfig->pArgs = malloc(8 * pConfig->dwNumberOfArgs);

	if (!pConfig->pArgs) {
		printf("[-] Unable To Allocate Memory For Arguments\n");
		return FALSE;
	}
	memset(pConfig->pArgs, 0x00, 8 * pConfig->dwNumberOfArgs);

	// Storing the Argument Values
	va_start(arg_list, dwArgCount);
	for (int i = 0; i < dwArgCount; i++) {

		((PUINT64)(pConfig->pArgs))[i] = va_arg(arg_list, UINT64);

	}

	return TRUE;
}
```

This function configures four as the minimum number of arguments, and if the number of arguments is odd, it is incremented by one. This is done to avoid complex steps in our assembly code to make sure that the stack will be aligned.

Windows x64 uses the `fastcall` calling convention. In the `fastcall` calling convention, the first four arguments for a function are stored in registers in the following order: `rcx`, `rdx`, `r8`, and `r9`. Any additional arguments are pushed to the stack from right to left. i.e. the 5th argument will be at the top of the stack, and the last argument will be at the bottom. If there are an odd number of arguments, an additional padding of 8 bytes is adjusted to maintain the stack alignment. Additionally, a 32-bit space must be reserved on the stack before calling the function, known as the shadow space. This space is used by the callee to store the first four arguments from the registers. Windows also requires the stack to be 16-byte aligned before a function is called. Basically, the value in `rsp` must be divisible by 16.

### ASM Walkthrough

The original blog explains the asm at this point, but I'll walk through it in my own way.

Before we dive in - what does DEFAULT REL mean in asm x64?

In x86-64 assembly (specifically for NASM), **`DEFAULT REL`** changes the assembler's default addressing mode for memory references from absolute to **RIP-relative** (Instruction Pointer-relative).

By default, NASM uses absolute addressing (`DEFAULT ABS`), which can cause issues in Position Independent Executables (PIE) or shared libraries because it hardcodes specific memory addresses. When **`DEFAULT REL`** is active:

* Bare memory references like `[variable]` are automatically treated as `[REL variable]`.
* The assembler calculates the offset from the current instruction pointer (RIP) to the target symbol, rather than using a fixed 64-bit absolute address.
* This is the standard practice for 64-bit Linux and Windows development to ensure code can be loaded at any memory address without modification.

why he used `DEFAULT REL` because he used relative addressing in his code which looks like this

<figure><img src="../.gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure>

now to understand this look at the screenshot below

<figure><img src="../.gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

so as per the example our bc = 15

<figure><img src="../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

and boom we did relative addressing. so now let's start with the code:

```nasm
BITS 64
DEFAULT REL
```

BITS 64 is to tell the assembler what arch we'll be using and DEFAULT REL is to make this code use relative addressing.

```nasm
STRUC Config
.pRopGadget: RESQ 1
.pTarget:	 RESQ 1
.dwArgCount: RESQ 1
.pRbx		 RESQ 1
.pArgs:		 RESQ 1
ENDSTRUC
```

In **NASM**, `struc Config` with `.pRopGadget: resq 1` defines a data structure template named **Config** containing a single field.

* **`struc Config`**: Begins the definition of a structure named `Config`.
* **`.pRopGadget`**: Defines a field (member) within the structure. The dot prefix indicates it is a local label relative to the structure.
* **`resq 1`**: Reserves space for **1 quadword**, which is **8 bytes** of memory. This is typically used for a 64-bit pointer or integer.

This directive creates a layout where `.pRopGadget` is an 8-byte offset from the start of the `Config` structure, allowing the assembly code to access this specific memory location using the structure's base address. same for the others.

```nasm
GLOBAL Spoof
SECTION .text
```

**GLOBAL** in the context of a `.text` section refers to the **visibility** or **scope** of symbols defined within that section. When a symbol is marked as **GLOBAL** (or `PUBLIC`/`EXTERN` depending on the assembler) within this section, it means:

* **Exported Symbol**: The function or variable is visible to other object files and the linker.
* **External Linkage**: Other modules in the program can call or reference this code.
* **Contrast with Local**: If a symbol is **LOCAL** (or `PRIVATE`), it is only visible within the current file and cannot be linked to from outside.

#### Spoof function

```nasm
	pop rdi								;pop the return Address and store in rdi register
	mov r10, rcx						;address of Config, which is passed as an argument in rcx
	mov r12d, [r10 + Config.dwArgCount]	;number of arguments are stored in r12
	sub r12d, 4							;no. of arguments on stack, as the first 4 are stored in registers
	mov r13, [r10 + Config.pArgs]		;args
	mov rcx, [r13]						;first arg
	mov rdx, [r13 + 8]					;second arg
	mov r8, [r13 + 16]					;third arg
	mov r9, [r13 + 24]					;fourth arg

	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
	; Loop To Move Arguments On The Stack
	;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

	lea r12, [r12 * 8]					;calculating the size of additional arguments
	sub rsp, r12						;making space on the stack
```

First we did pop where we saved our return address to whatever instruction was going to happen, then we put the address of config in r10 and how did config get into rcx? So basically in Windows x64 calling convention the first argument gets saved in rcx. To validate, let's see what the first arg was passed to `Spoof` in main.c code.

<figure><img src="../.gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

then we put all argcount in r12 and subtracted 4 because they all will go to stack space which is named as shadow space.

> What is Shadow Space
>
> **shadow space** (also known as **home space** or **spill space**) is a mandatory **32-byte** block allocated on the stack by the **caller** immediately before executing a `call` instruction. This space is reserved to allow the **callee** to optionally save the values of the first four register parameters (**RCX**, **RDX**, **R8**, and **R9**) for debugging or use as scratch space, regardless of how many arguments the function actually accepts.
>
> The allocation ensures that the stack remains **16-byte aligned**; since the `call` instruction pushes an 8-byte return address, the caller typically subtracts **40 bytes** from the stack pointer (`sub rsp, 40`) to provide the 32 bytes of shadow space plus 8 bytes for alignment. If a function requires more than four arguments, the additional arguments are pushed onto the stack **below** (at higher addresses than) the shadow space.

and then we moved all arguments in r13 by calling the `mov r13, [r10 + Config.pArgs]` so all arguments are saved in an 8 byte array with the help of (va\_start, va\_list and va\_arg).

**`va_list`**, **`va_start`**, and **`va_arg`** are essential components in C programming for handling **variadic functions** (functions that accept a variable number of arguments), defined within the `<stdarg.h>` header.

* **`va_list`**: An **object type** used to hold information needed by the macros to traverse the list of variable arguments. It acts as a pointer or handle to the current position in the argument stack.
* **`va_start`**: A macro that **initializes** the `va_list` variable. It must be called first to point the list to the first variable argument, requiring the name of the last fixed (named) parameter preceding the ellipsis (`...`).
* **`va_arg`**: A macro that **retrieves** the next argument from the list. It takes the `va_list` and a **type** as arguments, returning the current argument cast to that type and advancing the list pointer to the next item.

<figure><img src="../.gitbook/assets/image (71).png" alt=""><figcaption></figcaption></figure>

now we'll put our first 4 arguments in our registers which are rcx, rdx, r8 and r9. We are good to go if there are only 4, but what if there are 5 or 6? In our case of CreateThread it has 6 arguments, so we need to handle the extra ones.

```nasm
lea r12, [r12 * 8]			;calculating the size of additional arguments
sub rsp, r12				;making space on the stack
```

if u remember we have subtracted 4 from r12 so it just has the left over arguments if it has any, and in the CreateThread case it does have 6 arguments which means two of them are still left. So we multiplied that left over number by 8 coz one argument on the stack is 8 bytes, so it will become 16 in our case, and now we have created space for our left over arguments on the stack by subtracting r12 from rsp.

#### loop\_start

```nasm
	cmp r12, 0					;checking if the counter is zero
	jle loop_end
	mov r15, rsp				;copying stack pointer into temp variable
	add r15, r12				;address where argument needs to be written
	sub r15, 8
	mov rax, [r13 + 24 + r12]	;copying argument into temp variable
	mov [r15], rax				;writing argument on the stack
	sub r12, 8					;decrementing the counter
	jmp loop_start
```

first we'll check whether we do have any extra arguments or not, and if we don't have any, jmp to loop end. If we do then first copy the stack pointer into a temp register and then we first made it point normal, we added the value which is in r12 why? So that we can put arguments in a proper manner but remember that in the real stack it's still -16, we have added that value in this temp register so that we can set them in a manner.

> what manner: Basically the argument 5 should be at the top coz Windows takes arguments right to left, at the very bottom of the stack will be the argument which was called first. In our case it's argument 6 which will be at the bottom of the stack and highest in the address of the stack.

Now we'll do `sub r15, 8` so that we can put our 6th argument first and then 5 at the top as we discussed earlier. So now first we'll move the 6th argument to the stack.

First we subtracted 8 to make space for our 6th argument, then we put our 6th argument in rax with this, coz if u remember r13 holds our arguments `mov rax, [r13 + 24 + r12]` and then we moved the 6th argument to the stack by using this `mov [r15], rax`, how? coz r15 is rsp for now.

so I got the best demonstration, take it as that rsp is at 0 for now and then we subtracted 16 from it so we are at 16. Okay now what we do is we put rsp pointer to a temp register which is r15 and then we first add 16 to it so that now our temp rsp pointer is pointing at 0 basically, and then we do - 8 which will make our temp r15 which is our stack pointer to the highest address and then it will put it on 08, and then what we do is sub r12, 8 which makes it 8. Now again the loop starts and checks whether r12 which has the argument is empty or not, and if it's not then first mov rsp again to r15, now rsp is still at 16 and r15 is at 16 as well, then we add r12 to r15 and r12 is 8 now so our stack(r15) points at 8 and it subtracts to 8 again and r15 is again at 16 same as rsp and now it puts the 5th argument and boom.

and once it's all done it will move to loop\_end.

#### loop\_end

```nasm
	mov r13d, [r10 + Config.dwArgCount]	;storing the argument count in a non-volatile register
	sub rsp, 32							;shadow space
	mov rax, [r10 + Config.pRopGadget]	;copying return address to temp variable (Gadget's address)
	push rax							;pushing the return address on the stack
	lea rbx, [cleanup]					;setting the value of rbx. Rop gadget will jump to this address
	mov [r10 + Config.pRbx], rbx
	lea rbx, [r10 + Config.pRbx]
	mov r12, [r10 + Config.pTarget]
	jmp r12								;jumping to the target function
```

we have started by saving argument count in r13 and we have also created shadow space `sub rsp, 32` for those arguments. `mov rax, [r10 + Config.pRopGadget]` we have poured the rop gadget in rax and then we have pushed it as the top of the stack. Then we did `lea rbx, [cleanup]`, here we are setting up the value of rbx - where our rop gadget will jump to, this address is our cleanup function. But here is why we are doing it, because our rop gadget is basically `jmp QWORD PTR [rbx]`. As u can see that rbx is in \[] which means it's dereferencing it, so it's not taking the value which is in rbx, it's taking the address that rbx is pointing to, and it's not taking it, it's jumping on it. So it's jumping to the address where rbx is pointing, so to do that it does this:

```nasm
lea rbx, [cleanup]
mov [r10 + Config.pRbx], rbx
lea rbx, [r10 + Config.pRbx]
```

Now we'll put our target api function which we'll be using.

```nasm
lea rbx, [r10 + Config.pRbx]
mov r12, [r10 + Config.pTarget]
jmp r12
```

in this first we take out the function and then put it in r12 and jmp on that function address. So for example it's our VirtualAlloc, as of now we have set all our registers which were needed for VirtualAlloc, and then once the api is over and it hits its own ret, it will see our stack and it will see there our rop gadget which is pointing to our cleanup function. But for the system it will see that in the stack the return address points to kernel32 so it must be called by it and boom return address has been spoofed.

#### cleanup

```nasm
lea r13, [r13 * 8]
add rsp, r13
jmp rdi
```

in this first we are making our stack clean so first in our r13 it has the argument count and then we multiply it by 8 coz every argument has taken 8 bytes and then we add it to the stack and now the stack is normal. And in this `jmp rdi` it has the stored return address to our main.c coz whenever we call a function it first pushes the return address of that function to the stack.

and that's it.
