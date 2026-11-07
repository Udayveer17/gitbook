---
icon: pepper-hot
---

# PE Headers

## PE Headers

***

### PE Files

PE stands for Portable Executable, it’s a file format for executables used in Windows operating systems, it’s based on the `COFF` file format (Common Object File Format).

Not only `.exe` files are PE files, dynamic link libraries (`.dll`), Kernel modules (`.srv`), Control panel applications (`.cpl`) and many others are also PE files.

A PE file is a data structure that holds information necessary for the OS loader to be able to load that executable into memory and execute it.

***

### PE Structure

Recall the diagram below from the introductory module which shows a simplified structure of the PE format. Every header shown in the image is defined as a data structure that holds information about the PE file.

<figure><img src="../.gitbook/assets/image (80).png" alt=""><figcaption></figcaption></figure>

There are 9 predefined sections which are: .`text`, `.bss`, `.rdata`, `.data`, `.rsrc`, `.edata`, `.idata`, `.pdata` and `.debug` . (Not all are used by every application)

Most Common ones, however, are:

* .text: Executable Code Section
* .data, .rdata, .bss: Data Sections
* .rsrc: Resources Section
* .edata: Export data section
* .idata: import data section
* .debug: Debug information section

***

### DOS Header

#### Overview

The DOS header (also called the MS-DOS header) is a 64-byte-long structure that exists at the start of the PE file.

it’s not important for the functionality of PE files on modern Windows systems, however it’s there because of backward compatibility reasons.

This header makes the file an MS-DOS executable, so when it’s loaded on MS-DOS the DOS stub gets executed instead of the actual program.

Without this header, if you attempt to load the executable on MS-DOS it will not be loaded and will just produce a generic error.

#### Structure (IMAGE\_DOS\_HEADER)

DOS Header is first 64 bytes of PE, Now we can take a look at the structure and it’s contects by looking at `IMAGE_DOS_HEADER` which is definition from `winnt.h` :

```c
typedef struct _IMAGE_DOS_HEADER {      // DOS .EXE header
    WORD   e_magic;                     // Magic number
    WORD   e_cblp;                      // Bytes on last page of file
    WORD   e_cp;                        // Pages in file
    WORD   e_crlc;                      // Relocations
    WORD   e_cparhdr;                   // Size of header in paragraphs
    WORD   e_minalloc;                  // Minimum extra paragraphs needed
    WORD   e_maxalloc;                  // Maximum extra paragraphs needed
    WORD   e_ss;                        // Initial (relative) SS value
    WORD   e_sp;                        // Initial SP value
    WORD   e_csum;                      // Checksum
    WORD   e_ip;                        // Initial IP value
    WORD   e_cs;                        // Initial (relative) CS value
    WORD   e_lfarlc;                    // File address of relocation table
    WORD   e_ovno;                      // Overlay number
    WORD   e_res[4];                    // Reserved words
    WORD   e_oemid;                     // OEM identifier (for e_oeminfo)
    WORD   e_oeminfo;                   // OEM information; e_oemid specific
    WORD   e_res2[10];                  // Reserved words
    LONG   e_lfanew;                    // File address of new exe header
  } IMAGE_DOS_HEADER, *PIMAGE_DOS_HEADER;
```

The following picture shows contents of the DOS header in an actual PE file using PE-bear:

<figure><img src="../.gitbook/assets/image (81).png" alt=""><figcaption></figcaption></figure>

#### Important Members

**`e_magic`:** This is the first member of the DOS Header, it’s a WORD so it occupies 2 bytes, it’s usually called the magic number. It has a fixed value of `0x5A4D` or `MZ` in ASCII, and it serves as a signature that marks the file as an MS-DOS executable.

**`e_lfanew`:** This is the last member of the DOS header structure, it’s located at offset `0x3C` into the DOS header and it holds an offset to the start of the NT headers.At the end of DOS header and directly before DOS stub. The loader uses it to skip and go directly to NT header. The last DWORD before the DOS stub begins contains 00h 01h 00h 00h. Allowing for reverse byte order this gives us 00 00 01 00h which is the offset where the begins

> **NOTE:** DWORD ("double word") = 4 bytes or 32bit value, WORD = 2 bytes or 16bit value, sometimes you will also see dd for DWORD, dw for WORD and db for byte.

<figure><img src="../.gitbook/assets/image (82).png" alt=""><figcaption></figcaption></figure>

#### Retrieving the DOS Header

The following snippet demonstrates how to retrieve the DOS header.

```c
// Pointer to the structure
PIMAGE_DOS_HEADER pImgDosHdr = (PIMAGE_DOS_HEADER)pPE;
if (pImgDosHdr->e_magic != IMAGE_DOS_SIGNATURE){
	return -1;
}
```

***

### DOS Stub

The DOS stub is an MS-DOS program that prints an error message saying that the executable is not compatible with DOS then exits. This is what gets executed when the program is loaded in MS-DOS, the default error message is “This program cannot be run in DOS mode.”, however this message can be changed by the user during compile time.

That’s all we need to know about the DOS stub, we don’t really care about it

***

### Rich Header

So now we’ve seen the DOS Header and the DOS Stub, however there’s still a chunk of data we haven’t talked about lying between the DOS Stub and the start of the NT Headers.

<figure><img src="../.gitbook/assets/image (83).png" alt=""><figcaption></figcaption></figure>

This chunk of data is commonly referred to as the Rich Header, it’s an undocumented structure that’s only present in executables built using the Microsoft Visual Studio toolset.

This structure holds some metadata about the tools used to build the executable like their names or types and their specific versions and build numbers

sorry The Rich Header consists of a chunk of XORed data followed by a signature (`Rich` ) and a 32-bit checksum value that is the XOR key.

The encrypted data consists of a DWORD signature `DanS` , 3 zeroed-out DWORDs for padding, then pairs of DWORDS each pair representing an entry, and each entry holds a tool name, its build number and the number of times it’s been used.

In each DWORD pair the first pair holds the type ID or the product ID in the high WORD and the build ID in the low WORD, the second pair holds the use count.

***

### NT Headers (IMAGE\_NT\_HEADERS)

#### Overview

NT headers is a structure defined in `winnt.h` as `IMAGE_NT_HEADERS`, by looking at its definition we can see that it has three members, a `DWORD` signature, an `IMAGE_FILE_HEADER` structure called `FileHeader` and an `IMAGE_OPTIONAL_HEADER` structure called `OptionalHeader`.

It’s worth mentioning that this structure is defined in two different versions, one for 32-bit executables (Also named `PE32` executables) named `IMAGE_NT_HEADERS` and one for 64-bit executables (Also named `PE32+` executables) named `IMAGE_NT_HEADERS64`.

The main difference between the two versions is the used version of `IMAGE_OPTIONAL_HEADER` structure which has two versions, `IMAGE_OPTIONAL_HEADER32` for 32-bit executables and `IMAGE_OPTIONAL_HEADER64` for 64-bit executables.

#### Structure

```c
typedef struct _IMAGE_NT_HEADERS64 {
    DWORD Signature;
    IMAGE_FILE_HEADER FileHeader;
    IMAGE_OPTIONAL_HEADER64 OptionalHeader;
} IMAGE_NT_HEADERS64, *PIMAGE_NT_HEADERS64;

typedef struct _IMAGE_NT_HEADERS {
    DWORD Signature;
    IMAGE_FILE_HEADER FileHeader;
    IMAGE_OPTIONAL_HEADER32 OptionalHeader;
} IMAGE_NT_HEADERS32, *PIMAGE_NT_HEADERS32;
```

#### Retrieving the NT Headers

The `e_lfanew` member of the DOS header is an RVA to the `IMAGE_NT_HEADERS` structure. To reach the NT header, simply add the base address of the PE file in memory to the offset (`e_lfanew`). This is done in the following code snippet.

```c
// Pointer to the structure
PIMAGE_NT_HEADERS pImgNtHdrs = (PIMAGE_NT_HEADERS)(pPE + pImgDosHdr->e_lfanew);
if (pImgNtHdrs->Signature != IMAGE_NT_SIGNATURE) {
	return -1;
}
```

***

### File Header (IMAGE\_FILE\_HEADER)

#### Overview

Also called “The COFF File Header”, the File Header is a structure that holds some information about the PE file.

#### Structure

It’s defined as `IMAGE_FILE_HEADER` in `winnt.h`, here’s the definition:

```c
typedef struct _IMAGE_FILE_HEADER {
    WORD    Machine;
    WORD    NumberOfSections;
    DWORD   TimeDateStamp;
    DWORD   PointerToSymbolTable;
    DWORD   NumberOfSymbols;
    WORD    SizeOfOptionalHeader;
    WORD    Characteristics;
} IMAGE_FILE_HEADER, *PIMAGE_FILE_HEADER;
```

#### Retrieving the File Header

Since the file header is a member of the `IMAGE_NT_HEADERS` structure, it can be accessed using the following line of code.

```c
IMAGE_FILE_HEADER		ImgFileHdr	= pImgNtHdrs->FileHeader;
```

#### Members

It’s a simple structure with 7 members:

* **`Machine`:** This is a number that indicates the type of machine (CPU Architecture) the executable is targeting, this field can have a lot of values, but we’re only interested in two of them, `0x8864` for `AMD64` and `0x14c` for `i386`.
* **`NumberOfSections`:** This field holds the number of sections (or the number of section headers aka. the size of the section table.).
* **`TimeDateStamp`:** A `unix` timestamp that indicates when the file was created.
* **`PointerToSymbolTable` and `NumberOfSymbols`:** These two fields hold the file offset to the COFF symbol table and the number of entries in that symbol table, however they get set to `0` which means that no COFF symbol table(symbol table is **an array of fixed-size, 18-byte entries that store information about functions, variables, sections, and files within an object or executable file)** is present, this is done because the COFF debugging information is deprecated.
* **`SizeOfOptionalHeader`:** The size of the Optional Header.
* **`Characteristics`:** A flag that indicates the attributes of the file, these attributes can be things like the file being executable, the file being a system file and not a user program, and a lot of other things.

Here’s the File Header contents of an actual PE file:

<figure><img src="../.gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>

***

### Optional Header (IMAGE\_OPTIONAL\_HEADER)

#### Overview

The Optional Header is the most important header of the NT headers, the PE loader looks for specific information provided by that header to be able to load and run the executable. It’s called the optional header because some file types like object files don’t have it, however this header is essential for image files. It doesn’t have a fixed size, that’s why the `MAGE_FILE_HEADER.SizeOfOptionalHeader` member exists.

#### Structure

```c
typedef struct _IMAGE_OPTIONAL_HEADER64 {
    WORD        Magic;
    BYTE        MajorLinkerVersion;
    BYTE        MinorLinkerVersion;
    DWORD       SizeOfCode;
    DWORD       SizeOfInitializedData;
    DWORD       SizeOfUninitializedData;
    DWORD       AddressOfEntryPoint;
    DWORD       BaseOfCode;
    ULONGLONG   ImageBase;
    DWORD       SectionAlignment;
    DWORD       FileAlignment;
    WORD        MajorOperatingSystemVersion;
    WORD        MinorOperatingSystemVersion;
    WORD        MajorImageVersion;
    WORD        MinorImageVersion;
    WORD        MajorSubsystemVersion;
    WORD        MinorSubsystemVersion;
    DWORD       Win32VersionValue;
    DWORD       SizeOfImage;
    DWORD       SizeOfHeaders;
    DWORD       CheckSum;
    WORD        Subsystem;
    WORD        DllCharacteristics;
    ULONGLONG   SizeOfStackReserve;
    ULONGLONG   SizeOfStackCommit;
    ULONGLONG   SizeOfHeapReserve;
    ULONGLONG   SizeOfHeapCommit;
    DWORD       LoaderFlags;
    DWORD       NumberOfRvaAndSizes;
    IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];
} IMAGE_OPTIONAL_HEADER64, *PIMAGE_OPTIONAL_HEADER64;
```

#### Members

*   **`Magic`:** Microsoft documentation describes this field as an integer that identifies the state of the image, the documentation mentions three common values:

    * **`0x10B`:** Identifies the image as a `PE32` executable.
    * **`0x20B`:** Identifies the image as a `PE32+` executable.
    * **`0x107`:** Identifies the image as a ROM image.

    The value of this field is what determines whether the executable is 32-bit or 64-bit, `IMAGE_FILE_HEADER.Machine` is ignored by the Windows PE loader.
* **`MajorLinkerVersion` and `MinorLinkerVersion`:** The linker major and minor version numbers.
* **`SizeOfCode`:** This field holds the size of the code (`.text`) section, or the sum of all code sections if there are multiple sections.
* **`SizeOfInitializedData`:** This field holds the size of the initialized data (`.data`) section, or the sum of all initialized data sections if there are multiple sections.
* **`SizeOfUninitializedData`:** This field holds the size of the uninitialized data (`.bss`) section, or the sum of all uninitialized data sections if there are multiple sections.
* **`AddressOfEntryPoint`:** An RVA of the entry point when the file is loaded into memory. The documentation states that for program images this relative address points to the starting address and for device drivers it points to initialization function. For DLLs an entry point is optional, and in the case of entry point absence the `AddressOfEntryPoint` field is set to `0`.
* **`BaseOfCode`:** An RVA of the start of the code section when the file is loaded into memory.
* **`BaseOfData` (`PE32` Only):** An RVA of the start of the data section when the file is loaded into memory.
* **`ImageBase`:** This field holds the preferred address of the first byte of image when loaded into memory (the preferred base address), this value must be a multiple of 64K. Due to memory protections like ASLR, and a lot of other reasons, the address specified by this field is almost never used, in this case the PE loader chooses an unused memory range to load the image into, after loading the image into that address the loader goes into a process called the relocating where it fixes the constant addresses within the image to work with the new image base, there’s a special section that holds information about places that will need fixing if relocation is needed, that section is called the relocation section (`.reloc`).
* **`SectionAlignment`:** This field holds a value that gets used for section alignment in memory (in bytes), sections are aligned in memory boundaries that are multiples of this value. The documentation states that this value defaults to the page size for the architecture and it can’t be less than the value of `FileAlignment`.
* **`FileAlignment`:** Similar to `SectionAligment` this field holds a value that gets used for section raw data alignment **on disk** (in bytes), if the size of the actual data in a section is less than the `FileAlignment` value, the rest of the chunk gets padded with zeroes to keep the alignment boundaries. The documentation states that this value should be a power of 2 between 512 and 64K, and if the value of `SectionAlignment` is less than the architecture’s page size then the sizes of `FileAlignment` and `SectionAlignment` must match.
* **`MajorOperatingSystemVersion`, `MinorOperatingSystemVersion`, `MajorImageVersion`, `MinorImageVersion`, `MajorSubsystemVersion` and `MinorSubsystemVersion`:** These members of the structure specify the major version number of the required operating system, the minor version number of the required operating system, the major version number of the image, the minor version number of the image, the major version number of the subsystem and the minor version number of the subsystem respectively.
* **`Win32VersionValue`:** A reserved field that the documentation says should be set to `0`.
* **`SizeOfImage:`** The size of the image file (in bytes), including all headers. It gets rounded up to a multiple of `SectionAlignment` because this value is used when loading the image into memory.
* **`SizeOfHeaders`:** The combined size of the DOS stub, PE header (NT Headers), and section headers rounded up to a multiple of `FileAlignment`.
* **`CheckSum`:** A checksum of the image file, it’s used to validate the image at load time.
* **`Subsystem`:** This field specifies the Windows subsystem (if any) that is required to run the image, A complete list of the possible values of this field can be found on the [official Microsoft documentation](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format).
* **`DLLCharacteristics`:** This field defines some characteristics of the executable image file, like if it’s `NX` compatible and if it can be relocated at run time. I have no idea why it’s named `DLLCharacteristics`, it exists within normal executable image files and it defines characteristics that can apply to normal executable files. A complete list of the possible flags for `DLLCharacteristics` can be found on the [official Microsoft documentation](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format).
* **`SizeOfStackReserve`, `SizeOfStackCommit`, `SizeOfHeapReserve` and `SizeOfHeapCommit`:** These fields specify the size of the stack to reserve, the size of the stack to commit, the size of the local heap space to reserve and the size of the local heap space to commit respectively.
* **`LoaderFlags`:** A reserved field that the documentation says should be set to `0`.
* **`NumberOfRvaAndSizes` :** Size of the `DataDirectory` array.
* **`DataDirectory`:** An array of `IMAGE_DATA_DIRECTORY` structures

Let’s take a look at the Optional Header contents of an actual PE file.

<figure><img src="../.gitbook/assets/image (85).png" alt=""><figcaption></figcaption></figure>

#### Retrieving the Optional Header

Since the optional header is a member of the `IMAGE_NT_HEADERS` structure, it can be accessed using the following code.

```c
IMAGE_OPTIONAL_HEADER	ImgOptHdr = pImgNtHdrs->OptionalHeader;
if (ImgOptHdr.Magic != IMAGE_NT_OPTIONAL_HDR_MAGIC) {
	return -1;
}
```

***

### DataDirectory (IMAGE\_DATA\_DIRECTORY)

#### Overview

The last member of the `IMAGE_OPTIONAL_HEADER` structure was an array of `IMAGE_DATA_DIRECTORY` structures defined as follows:

```c
IMAGE_DATA_DIRECTORY DataDirectory[IMAGE_NUMBEROF_DIRECTORY_ENTRIES];
```

`IMAGE_NUMBEROF_DIRECTORY_ENTRIES` is a constant defined with the value `16`, meaning that this array can have up to 16 `IMAGE_DATA_DIRECTORY` entries:

```c
#define IMAGE_NUMBEROF_DIRECTORY_ENTRIES    16
```

#### Structure

An `IMAGE_DATA_DIRETORY` structure is defines as follows:

```c
typedef struct _IMAGE_DATA_DIRECTORY {
    DWORD   VirtualAddress;
    DWORD   Size;
} IMAGE_DATA_DIRECTORY, *PIMAGE_DATA_DIRECTORY;
```

It’s a very simple structure with only two members, first one being an RVA pointing to the start of the Data Directory and the second one being the size of the Data Directory.

#### What is a Data Directory?

So what is a Data Directory? Basically a Data Directory is a piece of data located within one of the sections of the PE file.

Data Directories contain useful information needed by the loader, an example of a very important directory is the Import Directory which contains a list of external functions imported from other libraries

#### Data Directory Entries

Here’s a list of Data Directories defined in `winnt.h`. (Each one of these values represents an index in the DataDirectory array):

```c
// Directory Entries

#define IMAGE_DIRECTORY_ENTRY_EXPORT          0   // Export Directory
#define IMAGE_DIRECTORY_ENTRY_IMPORT          1   // Import Directory
#define IMAGE_DIRECTORY_ENTRY_RESOURCE        2   // Resource Directory
#define IMAGE_DIRECTORY_ENTRY_EXCEPTION       3   // Exception Directory
#define IMAGE_DIRECTORY_ENTRY_SECURITY        4   // Security Directory
#define IMAGE_DIRECTORY_ENTRY_BASERELOC       5   // Base Relocation Table
#define IMAGE_DIRECTORY_ENTRY_DEBUG           6   // Debug Directory
//      IMAGE_DIRECTORY_ENTRY_COPYRIGHT       7   // (X86 usage)
#define IMAGE_DIRECTORY_ENTRY_ARCHITECTURE    7   // Architecture Specific Data
#define IMAGE_DIRECTORY_ENTRY_GLOBALPTR       8   // RVA of GP
#define IMAGE_DIRECTORY_ENTRY_TLS             9   // TLS Directory
#define IMAGE_DIRECTORY_ENTRY_LOAD_CONFIG    10   // Load Configuration Directory
#define IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT   11   // Bound Import Directory in headers
#define IMAGE_DIRECTORY_ENTRY_IAT            12   // Import Address Table
#define IMAGE_DIRECTORY_ENTRY_DELAY_IMPORT   13   // Delay Load Import Descriptors
#define IMAGE_DIRECTORY_ENTRY_COM_DESCRIPTOR 14   // COM Runtime descriptor
```

#### Accessing a Data Directory

The Data Directory can be accessed from the optional's header last member. This is an array of `IMAGE_DATA_DIRECTORY` meaning each element in the array is an `IMAGE_DATA_DIRECTORY` structure that references a special data directory.

The data directories can be accessed using the following line of code.

```c
IMAGE_DATA_DIRECTORY DataDir = ImgOptHdr.DataDirectory[#INDEX IN THE ARRAY#];
```

For example, retrieving the data directory of the export directory is done as follows:

```c
IMAGE_DATA_DIRECTORY ExpDataDir = ImgOptHdr.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT];
```

***

### Sections and Section Headers

#### Sections

Sections are the containers of the actual data of the executable file, they occupy the rest of the PE file after the headers, precisely after the section headers.

Some sections have special names that indicate their purpose, we’ll go over some of them, and a full list of these names can be found on the [official Microsoft documentation](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format) under the “Special Sections” section.

* **`.text`:** Contains the executable code of the program.
* **`.data`:** Contains the initialized data.
* **`.bss`:** Contains uninitialized data.
* **`.rdata`:** Contains read-only initialized data.
* **`.edata`:** Contains the export tables.
* **`.idata`:** Contains the import tables.
* **`.reloc`:** Contains image relocation information.
* **`.rsrc`:** Contains resources used by the program, these include images, icons or even embedded binaries.
* **`.tls`:** (**T**hread **L**ocal **S**torage), provides storage for every executing thread of the program.

#### Section Headers (IMAGE\_SECTION\_HEADER)

After the Optional Header and before the sections comes the Section Headers. These headers contain information about the sections of the PE file. and

A Section Header is a structure named `IMAGE_SECTION_HEADER` defined in `winnt.h` as follows:

```c
typedef struct _IMAGE_SECTION_HEADER {
    BYTE    Name[IMAGE_SIZEOF_SHORT_NAME];
    union {
            DWORD   PhysicalAddress;
            DWORD   VirtualSize;
    } Misc;
    DWORD   VirtualAddress;
    DWORD   SizeOfRawData;
    DWORD   PointerToRawData;
    DWORD   PointerToRelocations;
    DWORD   PointerToLinenumbers;
    WORD    NumberOfRelocations;
    WORD    NumberOfLinenumbers;
    DWORD   Characteristics;
} IMAGE_SECTION_HEADER, *PIMAGE_SECTION_HEADER;
```

#### Section Header Members

* **`Name`:** First field of the Section Header, a byte array of the size `IMAGE_SIZEOF_SHORT_NAME` that holds the name of the section. `IMAGE_SIZEOF_SHORT_NAME` has the value of `8` meaning that a section name can’t be longer than 8 characters. For longer names the official documentation mentions a work-around by filling this field with an offset in the string table, however executable images do not use a string table so this limitation of 8 characters holds for executable images.
* **`PhysicalAddress` or `VirtualSize`:** A `union` defines multiple names for the same thing, this field contains the total size of the section when it’s loaded in memory.
* **`VirtualAddress`:** The documentation states that for executable images this field holds the address of the first byte of the section relative to the image base when loaded in memory, and for object files it holds the address of the first byte of the section before relocation is applied.
* **`SizeOfRawData`:** This field contains the size of the section on disk, it must be a multiple of `IMAGE_OPTIONAL_HEADER.FileAlignment`. `SizeOfRawData` and `VirtualSize` can be different, we’ll discuss the reason for this later in the post.
* **`PointerToRawData`:** A pointer to the first page of the section within the file, for executable images it must be a multiple of `IMAGE_OPTIONAL_HEADER.FileAlignment`.
* **`PointerToRelocations`:** A file pointer to the beginning of relocation entries for the section. It’s set to `0` for executable files.
* **`PointerToLineNumbers`:** A file pointer to the beginning of COFF line-number entries for the section. It’s set to `0` because COFF debugging information is deprecated.
* **`NumberOfRelocations`:** The number of relocation entries for the section, it’s set to `0` for executable images.
* **`NumberOfLinenumbers`:** The number of COFF line-number entries for the section, it’s set to `0` because COFF debugging information is deprecated.
* **`Characteristics`:** Flags that describe the characteristics of the section. These characteristics are things like if the section contains executable code, contains initialized/uninitialized data, can be shared in memory. A complete list of section characteristics flags can be found on the [official Microsoft documentation](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format).

#### SizeOfRawData vs VirtualSize

`SizeOfRawData` and `VirtualSize` can be different, and this can happen for multiple of reasons.

`SizeOfRawData` must be a multiple of `IMAGE_OPTIONAL_HEADER.FileAlignment`, so if the section size is less than that value the rest gets padded and `SizeOfRawData` gets rounded to the nearest multiple of `IMAGE_OPTIONAL_HEADER.FileAlignment`.

However when the section is loaded into memory it doesn’t follow that alignment and only the actual size of the section is occupied.

In this case `SizeOfRawData` will be greater than `VirtualSize`

The opposite can happen as well.

If the section contains uninitialized data, these data won’t be accounted for on disk, but when the section gets mapped into memory, the section will expand to reserve memory space for when the uninitialized data gets later initialized and used.

This means that the section on disk will occupy less than it will do in memory, in this case `VirtualSize` will be greater than `SizeOfRawData`.

#### Retrieving The IMAGE\_SECTION\_HEADER Structure

The `IMAGE_SECTION_HEADER` structure is stored in an array within the PE file's headers. To access the first element, skip past the `IMAGE_NT_HEADERS` since the sections are located immediately after the NT headers. The following snippet shows how to retrieve the `IMAGE_SECTION_HEADER` structure, where `pImgNtHdrs` is a pointer to `IMAGE_NT_HEADERS` structure.

```c
PIMAGE_SECTION_HEADER pImgSectionHdr = (PIMAGE_SECTION_HEADER)(((PBYTE)pImgNtHdrs) + sizeof(IMAGE_NT_HEADERS));
```

#### 10.6 Looping Through The Array

Looping through the array requires the array size which can be retrieved from the `IMAGE_FILE_HEADER.NumberOfSections` member. The subsequent elements in the array are located at an interval of `sizeof(IMAGE_SECTION_HEADER)` from the current element.

```c
PIMAGE_SECTION_HEADER pImgSectionHdr = (PIMAGE_SECTION_HEADER)(((PBYTE)pImgNtHdrs) + sizeof(IMAGE_NT_HEADERS));

for (size_t i = 0; i < pImgNtHdrs->FileHeader.NumberOfSections; i++) {
	// pImgSectionHdr is a pointer to section 1
	pImgSectionHdr = (PIMAGE_SECTION_HEADER)((PBYTE)pImgSectionHdr + (DWORD)sizeof(IMAGE_SECTION_HEADER));
	// pImgSectionHdr is a pointer to section 2
}
```

***

### Export Table

#### Export Table Structure (IMAGE\_EXPORT\_DIRECTORY)

The export table is a structure defined as `IMAGE_EXPORT_DIRECTORY` which is shown below.

```c
typedef struct _IMAGE_EXPORT_DIRECTORY {
    DWORD   Characteristics;
    DWORD   TimeDateStamp;
    WORD    MajorVersion;
    WORD    MinorVersion;
    DWORD   Name;
    DWORD   Base;
    DWORD   NumberOfFunctions;
    DWORD   NumberOfNames;
    DWORD   AddressOfFunctions;     // RVA from base of image
    DWORD   AddressOfNames;         // RVA from base of image
    DWORD   AddressOfNameOrdinals;  // RVA from base of image
} IMAGE_EXPORT_DIRECTORY, *PIMAGE_EXPORT_DIRECTORY;
```

#### Retrieving The Export Table

The `IMAGE_EXPORT_DIRECTORY` structure is used to store information about the functions and data that are exported from a PE file. This information is stored in the data directory array with the index `IMAGE_DIRECTORY_ENTRY_EXPORT`. To fetch it from the `IMAGE_OPTIONAL_HEADER` structure:

```c
PIMAGE_EXPORT_DIRECTORY pImgExportDir = (PIMAGE_EXPORT_DIRECTORY)(pPE + ImgOptHdr.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress);
```

Where `pPE` is the base address of the loaded PE in memory and `ImgOptHdr` is the `IMAGE_OPTIONAL_HEADER` structure previously calculated.

#### Export Table Important Members

The most important members of the export table are the following:

* `NumberOfFunctions` - Specifies the number of functions that are exported by the PE file.
* `NumberOfNames` - Specifies the number of names that are exported by the PE file.
* `AddressOfFunctions` - Specifies the address of an array of addresses of the exported functions.
* `AddressOfNames` - Specifies the address of an array of addresses of the names of the exported functions.
* `AddressOfNameOrdinals` - Specifies the address of an array of ordinal numbers for the exported functions.

***

### Imports

#### Import Directory Table (IMAGE\_IMPORT\_DESCRIPTOR)

The Import Directory Table is a Data Directory located at the beginning of the `.idata` section.

It consists of an array of `IMAGE_IMPORT_DESCRIPTOR` structures, each one of them is for a DLL.

It doesn’t have a fixed size, so the last `IMAGE_IMPORT_DESCRIPTOR` of the array is zeroed-out (NULL-Padded) to indicate the end of the Import Directory Table.

`IMAGE_IMPORT_DESCRIPTOR` is defined as follows:

```c
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
    union {
        DWORD   Characteristics;
        DWORD   OriginalFirstThunk;
    } DUMMYUNIONNAME;
    DWORD   TimeDateStamp;
    DWORD   ForwarderChain;
    DWORD   Name;
    DWORD   FirstThunk;
} IMAGE_IMPORT_DESCRIPTOR;
typedef IMAGE_IMPORT_DESCRIPTOR UNALIGNED *PIMAGE_IMPORT_DESCRIPTOR;
```

* **`OriginalFirstThunk`:** RVA of the ILT.
* **`TimeDateStamp`:** A time date stamp, that’s initially set to `0` if not bound and set to `1` if bound. In case of an unbound import the time date stamp gets updated to the time date stamp of the DLL after the image is bound. In case of a bound import it stays set to `1` and the real time date stamp of the DLL can be found in the Bound Import Directory Table in the corresponding `IMAGE_BOUND_IMPORT_DESCRIPTOR` .
* **`ForwarderChain`:** The index of the first forwarder chain reference. This is something responsible for DLL forwarding. (DLL forwarding is when a DLL forwards some of its exported functions to another DLL.)
* **`Name`:** An RVA of an ASCII string that contains the name of the imported DLL.
* **`FirstThunk`:** RVA of the IAT.

#### Import Lookup Table (ILT)

Sometimes people refer to it as the Import Name Table (INT).

Every imported DLL has an Import Lookup Table.

`IMAGE_IMPORT_DESCRIPTOR.OriginalFirstThunk` holds the RVA of the ILT of the corresponding DLL.

The ILT is essentially a table of names or references, it tells the loader which functions are needed from the imported DLL.

The ILT consists of an array of 32-bit numbers (for PE32) or 64-bit numbers for (PE32+), the last one is zeroed-out to indicate the end of the ILT.

Each entry of these entries encodes information as follows:

* **Bit 31/63 (most significant bit)**: This is called the Ordinal/Name flag, it specifies whether to import the function by name or by ordinal.
* **Bits 15-0:** If the Ordinal/Name flag is set to `1` these bits are used to hold the 16-bit ordinal number that will be used to import the function, bits 30-15/62-15 for PE32/PE32+ must be set to `0`.
* **Bits 30-0:** If the Ordinal/Name flag is set to `0` these bits are used to hold an RVA of a Hint/Name table.

**Hint/Name Table (IMAGE\_IMPORT\_BY\_NAME)**

A Hint/Name table is a structure defined in `winnt.h` as `IMAGE_IMPORT_BY_NAME`:

```c
typedef struct _IMAGE_IMPORT_BY_NAME {
    WORD    Hint;
    CHAR   Name[1];
} IMAGE_IMPORT_BY_NAME, *PIMAGE_IMPORT_BY_NAME;
```

* **`Hint`:** A word that contains a number, this number is used to look-up the function, that number is first used as an index into the export name pointer table, if that initial check fails a binary search is performed on the DLL’s export name pointer table.
* **`Name`:** A null-terminated string that contains the name of the function to import.

#### Import Address Table (IMAGE\_IMPORT\_DESCRIPTOR)

The import address table is an array of `IMAGE_IMPORT_DESCRIPTOR` structures with each one being for a DLL file that contains the functions that were used from these DLLs.

**Import Address Table Structure**

The `IMAGE_IMPORT_DESCRIPTOR` structure is also not officially documented by Microsoft although it is defined in the [Winnt.h Header File](https://learn.microsoft.com/en-us/windows/win32/api/winnt/) as follows:

```c
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
    union {
        DWORD   Characteristics;
        DWORD   OriginalFirstThunk;
    } DUMMYUNIONNAME;
    DWORD   TimeDateStamp;
    DWORD   ForwarderChain;
    DWORD   Name;
    DWORD   FirstThunk;
} IMAGE_IMPORT_DESCRIPTOR;
```

**Retrieving The Import Address Table**

To fetch the import address table from the `IMAGE_OPTIONAL_HEADER` structure:

```c
IMAGE_IMPORT_DESCRIPTOR* pImgImpDesc = (PIMAGE_IMPORT_DESCRIPTOR)(pPE + ImgOptHdr.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress);
```

Where `pPE` is the base address of the loaded PE in memory and `ImgOptHdr` is the `IMAGE_OPTIONAL_HEADER` structure previously calculated.

#### Bound Imports

A bound import essentially means that the import table contains fixed addresses for the imported functions. These addresses are calculated and written during compile time by the linker.

Using bound imports is a speed optimization, it reduces the time needed by the loader to resolve function addresses and fill the IAT, however if at run-time the bound addresses do not match the real ones then the loader will have to resolve these addresses again and fix the IAT.

#### Bound Import Data Directory (IMAGE\_BOUND\_IMPORT\_DESCRIPTOR)

The Bound Import Data Directory is similar to the Import Directory Table, however as the name suggests, it holds information about the bound imports.

It consists of an array of `IMAGE_BOUND_IMPORT_DESCRIPTOR` structures, and ends with a zeroed-out `IMAGE_BOUND_IMPORT_DESCRIPTOR`.

`IMAGE_BOUND_IMPORT_DESCRIPTOR` is defined as follows:

```c
typedef struct _IMAGE_BOUND_IMPORT_DESCRIPTOR {
    DWORD   TimeDateStamp;
    WORD    OffsetModuleName;
    WORD    NumberOfModuleForwarderRefs;
// Array of zero or more IMAGE_BOUND_FORWARDER_REF follows
} IMAGE_BOUND_IMPORT_DESCRIPTOR,  *PIMAGE_BOUND_IMPORT_DESCRIPTOR;
```

* **`TimeDateStamp`:** The time date stamp of the imported DLL.
* **`OffsetModuleName`:** An offset to a string with the name of the imported DLL. It’s an offset from the first `IMAGE_BOUND_IMPORT_DESCRIPTOR`
* **`NumberOfModuleForwarderRefs`:** The number of the `IMAGE_BOUND_FORWARDER_REF` structures that immediately follow this structure. `IMAGE_BOUND_FORWARDER_REF` is a structure that’s identical to `IMAGE_BOUND_IMPORT_DESCRIPTOR`, the only difference is that the last member is reserved.

***

### Other Data Directory Structures

#### IMAGE\_TLS\_DIRECTORY

`IMAGE_TLS_DIRECTORY` - This structure is used to store information about [Thread-Local Storage](https://learn.microsoft.com/en-us/cpp/c-language/thread-local-storage?view=msvc-170) (TLS) data in the PE file. It is important to be aware of how to retrieve this structure from the `IMAGE_OPTIONAL_HEADER` structure at this time; further details will be provided in subsequent modules when necessary.

```c
PIMAGE_TLS_DIRECTORY pImgTlsDir  = (PIMAGE_TLS_DIRECTORY)(pPE + ImgOptHdr.DataDirectory[IMAGE_DIRECTORY_ENTRY_TLS].VirtualAddress);
```

#### IMAGE\_RUNTIME\_FUNCTION\_ENTRY

`IMAGE_RUNTIME_FUNCTION_ENTRY` - This structure is used to store information about a runtime function in the PE file. A runtime function is a function that is called by the Windows operating system's exception handling mechanism to execute the exception handling code for an exception. It is important to be aware of how to retrieve this structure from the `IMAGE_OPTIONAL_HEADER` structure at this time; further details will be provided in subsequent modules when necessary.

```c
PIMAGE_RUNTIME_FUNCTION_ENTRY pImgRunFuncEntry = (PIMAGE_RUNTIME_FUNCTION_ENTRY)(pPE + ImgOptHdr.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXCEPTION].VirtualAddress);
```

#### IMAGE\_BASE\_RELOCATION

`IMAGE_BASE_RELOCATION` - This structure is used to store information about the base relocations in the PE file. Base relocations are used to fix up the addresses of imported functions and variables in a PE file when it is loaded into memory at an address that differs from the address at which it was linked. It is important to be aware of how to retrieve this structure from the `IMAGE_OPTIONAL_HEADER` structure at this time; further details will be provided in subsequent modules when necessary.

```c
PIMAGE_BASE_RELOCATION pImgBaseReloc = (PIMAGE_BASE_RELOCATION)(pPE + ImgOptHdr.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC].VirtualAddress);
```
