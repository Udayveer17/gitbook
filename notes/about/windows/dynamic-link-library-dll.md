---
icon: book-spine
---

# Dynamic-link library (DLL)

## What is a DLL File?

A Dynamic Link Library (DLL) is a binary file format that stores compiled code and data, designed for dynamic linking during runtime. It enables the shared use of functions and resources by multiple programs on the Windows operating system.

DLL plays a crucial role in promoting code reusability, reducing redundancy, and optimizing memory. These files contain executable code, such as functions and procedures, as well as shared data that can be utilized by various applications.

DLLs support versioning to manage compatibility and may have dependencies on other DLLs. Their file format includes compiled machine code and additional information, contributing to efficient and modular software development on the Windows platform.

DLLs are crafted by developers who write custom code to execute specific functions. This code is typically written in C, C++or C# and then compiled in a DLL file.

### DLL Working:

We now know what is DLL and why it is used. Let’s move towards how a DLL works when you click a program that requires it step by step.

#### Loading DLL in Memory

When an executable (.exe) file is launched, the operating system (OS) takes the necessary steps to launch and run the associated program. As part of this process, the OS loads the program into the computer’s memory, allocating the required resources for its execution. In cases where the program relies on external code modules known as Dynamic Link Libraries (DLLs), the OS searches for these DLLs in various predefined locations.

These locations typically include the system directory, the program directory, and the current directory. Once the necessary DLL is found, the OS loads it into the memory space of the program. This allows the program to access the functions and resources provided by the DLL, facilitating modular code organization and reuse across multiple applications

#### Linking of DLLs

When you load a DLL in an application, there are two methods of linking to call the exported DLL functions. They are _**load-time dynamic linking**_ and _**run-time dynamic linking.**_

### Load Time Dynamic Linking

***

Before I jump into how Load-Time Dynamic Linking works, let me quickly explain what a **linker** is because you need to understand that first.

**How the linker works specifically for Load-Time Dynamic Linking.**

In Load-Time Linking, you write your code normally. You just call functions from DLLs directly like `MessageBoxA(NULL, "Hello", "Title", MB_OK);` without any extra fuss. When the linker processes your code during the build phase, it sees this call to `MessageBoxA` and realizes this function doesn't exist in your own code. The linker knows it has to come from some external DLL.

Instead of copying the actual code of `MessageBoxA` into your executable, the linker does something smarter. It creates a table inside your executable called the **Import Address Table** or IAT for short. Think of this table as a shopping list. For every external function your program uses, the linker adds one entry to this list. Each entry has the function name like `MessageBoxA` and the DLL name like `user32.dll`. But here's the catch the linker can't put the actual memory address of the function in this table because the DLL hasn't been loaded into memory yet. Nobody knows where `user32.dll` will end up in memory when your program actually runs. So the linker just leaves the address part empty.

Your executable file gets a tiny bit bigger because it now has this table inside it, but it's not a big deal because the table just has names and references, not actual code. Once the linker finishes creating this table and packs it into your executable, its work is done. The linker doesn't do anything else after this point. It only runs during the build phase.

Now here's what happens when you actually double-click your executable to run it. The Windows operating system has a component called the OS Loader that takes over. The OS Loader reads your executable file, finds the Import Address Table that the linker created, and starts going through it entry by entry. For each entry, it sees "I need MessageBoxA from user32.dll". So the OS Loader goes to your hard drive, finds user32.dll, loads it into your program's memory, finds the MessageBoxA function inside that DLL, and writes the actual memory address into that empty spot in the Import Address Table. The OS Loader does this for every single entry in the table.

Only after the OS Loader has resolved every single entry in the Import Address Table does it finally let your program start running. This means by the time your `main()` function executes, every DLL function you need is already loaded and every address is already filled in. When your code calls `MessageBoxA()`, it just looks at the Import Address Table, sees the address is already there, and jumps directly to that location. This makes everything really fast because there's no extra searching or loading happening during the call.

There are some important consequences to how this works. Because the linker didn't copy the DLL code into your executable, your file stays small and multiple programs can share the same DLL file on disk and in memory, which saves space. Also, because the linker left the addresses empty and the OS Loader fills them at startup, you can update the DLL without recompiling your program. Just drop in the new DLL file and the OS Loader will find the updated functions the next time your program runs.

But there's a downside too. Since the OS Loader resolves everything before your program starts, if a DLL is missing, your program simply fails to start. You get an error message and that's it. You don't get any chance to handle this gracefully because the failure happens before your code even runs.

***

### Run Time Dynamic Linking

Now let me explain Run-Time Dynamic Linking. This one works completely differently from Load-Time Linking because you're in charge of loading the DLL instead of the OS Loader doing it automatically.

In Run-Time Linking, you don't call the DLL function directly in your code. Instead, you write extra code to manually load the DLL and grab the function's address while your program is already running. You use two special Windows functions called `LoadLibrary()` and `GetProcAddress()`. These are always available to you.

When the linker processes your code, it sees you're calling `LoadLibrary` and `GetProcAddress`, but it doesn't see you directly calling `MessageBoxA`. Because you never directly called `MessageBoxA`, the linker doesn't bother adding it to the Import Address Table. The linker only makes sure `LoadLibrary` and `GetProcAddress` themselves are available. This means your executable file is smaller because it doesn't have an entry for `MessageBoxA` in that table.

Now when your program runs and reaches the point where it needs that DLL function, your code calls `LoadLibrary("user32.dll")`. This tells Windows to go find user32.dll on your hard drive and load it into your program's memory right at that moment. `LoadLibrary` gives you back a handle, which is basically a reference number for the loaded DLL. Then you pass this handle along with the function name `"MessageBoxA"` to `GetProcAddress`. This function looks inside the loaded DLL, finds MessageBoxA, and gives you back its exact memory address. You store this address in what's called a function pointer, and then you call the function using that pointer.

The beauty of this approach is that you're in total control. If user32.dll doesn't exist on the system, `LoadLibrary` will fail and your code can check for that and do something sensible. You could show a friendly error message, log what happened, or even try loading a different DLL instead. Your program doesn't crash and die it can keep running with reduced features. You can also load different DLLs based on what the user wants or what hardware they have. For example, a game might load DirectX11 or OpenGL depending on what graphics card you have.

But of course there are tradeoffs. Your code gets more complicated because you have to write error handling for `LoadLibrary`, check if `GetProcAddress` worked, manage function pointers carefully, and maybe even call `FreeLibrary` to unload the DLL when you're done. Also, while calling functions through that pointer is almost as fast as Load-Time Linking once everything is loaded, the first call to `LoadLibrary` takes a little time. And there's a tiny bit of overhead from the extra pointer dereference. Debugging is also trickier because if you type the function name wrong in `GetProcAddress`, the compiler won't catch it. Your program will only break at runtime when it tries to find that function.

So to sum it up, Run-Time Linking gives you more flexibility and control at the cost of more complex code and a tiny bit of extra overhead.

***

### What Actually Gets Copied into Virtual Memory?

When the OS Loader loads a DLL, it loads **the entire DLL** into your program's virtual memory, not just one function. The whole DLL file gets mapped into memory all of its code, all of its data, everything. Your program doesn't just get `MessageBoxA` it gets the entire `user32.dll` with all its functions like `CreateWindow`, `GetMessage`, `SendMessage`, and everything else that's inside that DLL.

Think of it like this when the OS Loader sees your program needs `MessageBoxA` from `user32.dll`, it doesn't just grab that one function. It grabs the whole DLL file, loads the entire thing into memory, and then your program can call ANY function from that DLL. You just happen to be using `MessageBoxA` right now, but you could also call `CreateWindow` from the same DLL if you wanted to because the whole DLL is sitting there in memory.

Think of it like this when the OS Loader sees your program needs `MessageBoxA` from `user32.dll`, it doesn't just grab that one function. It grabs the whole DLL file, loads the entire thing into memory, and then your program can call ANY function from that DLL. You just happen to be using `MessageBoxA` right now, but you could also call `CreateWindow` from the same DLL if you wanted to because the whole DLL is sitting there in memory.

***

### The Complete Step-by-Step Process

Let me walk through the entire process from start to finish with all the parts included:

**Step 1: You Write Your Code**

You write `MessageBoxA(NULL, "Hello", "Title", MB_OK);` in your program.

**Step 2: The Linker Does Its Work During Build**

The linker sees you're calling `MessageBoxA`. It creates the Import Address Table (IAT) in your executable. The IAT has an entry for `MessageBoxA` that says:

* Function name: `MessageBoxA`
* DLL name: `user32.dll`
* Address: \[EMPTY linker leaves this blank because it doesn't know where the DLL will be loaded]

**Step 3: You Double-Click Your Executable to Run It**

The Windows OS Loader takes over and starts reading your executable file.

**Step 4: The OS Loader Finds the IAT**

The OS Loader looks inside your executable, finds the Import Address Table, and sees the entry that says "This program needs `MessageBoxA` from `user32.dll`."

**Step 5: The OS Loader Loads the Entire DLL into Virtual Memory**

The OS Loader goes to your hard drive, finds `user32.dll`, reads the entire file, and loads the whole DLL into your program's virtual memory space. This means every single function in `user32.dll` all the code, all the data, everything is now sitting in your program's memory.

**Step 6: The OS Loader Fills in the IAT**

Now the OS Loader knows exactly where in memory the `MessageBoxA` function is because the whole DLL is loaded. The OS Loader goes back to the Import Address Table and fills in that empty address space with the actual memory address of `MessageBoxA` inside the loaded DLL. For example, if the DLL was loaded at address `0x70000000` and `MessageBoxA` is `0x1000` bytes into the DLL, the OS Loader puts `0x70001000` into the IAT.

**Step 7: The OS Loader Finishes and Your Program Starts**

Only after the OS Loader has resolved every single entry in the IAT does it finally let your program start running. Windows transfers control to your `main()` function and your code begins executing.

**Step 8: Your Program Calls the Function**

When your code hits `MessageBoxA(NULL, "Hello", "Title", MB_OK)`, here's exactly what happens:

* Your program looks at the IAT entry for `MessageBoxA`
* The IAT says "The address is 0x70001000" (this was filled in by the OS Loader in Step 6)
* Your program tells the CPU "jump to address 0x70001000 and start executing the code there"

**Step 9: The CPU Executes the DLL's Code**

The CPU jumps to that memory address `0x70001000`. Remember, this address is inside your program's virtual memory where the DLL was loaded. The CPU starts executing the machine code of `MessageBoxA` that came from `user32.dll`. The DLL code runs, does whatever it needs to do (draws a message box on screen, waits for user to click OK, etc.), and then finishes its work.

**Step 10: Control Returns to Your Program**

Once the DLL function finishes, the CPU executes a return instruction. This tells the CPU to go back to where it came from back to your program's code right after the call to `MessageBoxA`. Your program continues running from that point.

***

**From "How Does a Linker Work" type articles:**

[https://www.geeksforgeeks.org/memory-layout-of-c-program/](https://www.geeksforgeeks.org/memory-layout-of-c-program/)

[https://www.learncpp.com/cpp-tutorial/introduction-to-linkers/](https://www.learncpp.com/cpp-tutorial/introduction-to-linkers/)

[https://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Mips/link.html](https://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Mips/link.html)

[https://stackoverflow.com/questions/3322911/what-do-linkers-do](https://stackoverflow.com/questions/3322911/what-do-linkers-do)

***

**From Windows Loader/DLL Articles:**

[https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order)

[https://learn.microsoft.com/en-us/windows/win32/dlls/load-time-dynamic-linking](https://learn.microsoft.com/en-us/windows/win32/dlls/load-time-dynamic-linking)

[https://learn.microsoft.com/en-us/windows/win32/dlls/run-time-dynamic-linking](https://learn.microsoft.com/en-us/windows/win32/dlls/run-time-dynamic-linking)

[https://devblogs.microsoft.com/oldnewthing/20150212-00/?p=44633](https://devblogs.microsoft.com/oldnewthing/20150212-00/?p=44633)

***

**PE Format / IAT Deep Dives:**

[https://learn.microsoft.com/en-us/windows/win32/debug/pe-format](https://learn.microsoft.com/en-us/windows/win32/debug/pe-format)

[https://0xrick.github.io/win-internals/pe1/](https://0xrick.github.io/win-internals/pe1/)

[https://www.ired.team/offensive-security/code-injection-process-injection/binary-exploitation/pe-file-structure-and-import-address-table-iat](https://www.ired.team/offensive-security/code-injection-process-injection/binary-exploitation/pe-file-structure-and-import-address-table-iat)

[https://www.matteomalvica.com/blog/2020/04/13/windows-pe-format-internals/](https://www.matteomalvica.com/blog/2020/04/13/windows-pe-format-internals/)

***

**Practical / Code Examples:**

[https://github.com/nikoskalogridis/PE-Loader](https://github.com/nikoskalogridis/PE-Loader)

[https://github.com/fancycode/MemoryModule](https://github.com/fancycode/MemoryModule)
