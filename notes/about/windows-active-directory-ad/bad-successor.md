---
description: In This we will learn how BadSuccessor work
icon: bed-empty
---

# Bad-successor

> To Understand BadSuccessor first we have to understand what is dMSA&#x20;

## **Delegated Managed Service Accounts (dMSA)**

### **What is dMSA?**

A new Windows Server 2025 service account type that improves security over traditional and gMSA accounts by enforcing machine-based authentication.

***

## **Why Traditional Service Accounts Are Weak**

* Static passwords, rarely rotated
* Easy targets for theft (Kerberoasting)
* Stolen creds work from _any_ machine
* Require manual password management

***

## **How dMSA Works**

* **Machine-bound authentication:** AD checks _who_ you are and _which machine_ you’re on
* **Authorized machines only:** You specify allowed hosts
* **Automatic password management:** Fully randomized and auto-rotated
* **Old password disabled** when converting from traditional accounts

***

## **Machine-Based Authentication (Simple)**

**Traditional:**\
Only checks identity -anyone with the password can log in anywhere.

**Machine-Based:**\
Checks identity **and** machine - creds only work on approved hosts.\
Even if an attacker steals the password, they can’t use it from their machine.

***

## **dMSA vs gMSA (Quick Comparison)**

#### **gMSA**

* Auto password rotation
* Can run across multiple servers
* Weak point: stolen gMSA password works on _any_ machine using that gMSA

#### **dMSA**

* Does everything gMSA does **plus:**
  * Locks authentication to specific machines
  * Uses Credential Guard
  * Stolen creds only work on the original machine
* Even if Server A is hacked, attacker cannot reuse the dMSA credential anywhere else

***

## **How dMSA Is Created**

1. **New Account** – fresh dMSA for new services
2. **Convert Existing Account** – replaces old service accounts

***

## **Migration Behavior**

**Step 1 Old Password Blocked:**\
Old static password stops working immediately.

**Step 2  LSA Redirection:**\
LSA intercepts old login attempts - forces dMSA machine-based auth.

**Step 3 Permissions Preserved:**\
New dMSA keeps all the same access rights as the original service account.

***

## **Automatic Machine Discovery**

During migration, AD auto-detects all machines using the old account and builds the authorized machine list for dMSA.\
No need to manually track usage.

***

## **How dMSA Secrets Work**

* The “secret” is a randomized credential derived from the machine’s identity
* Used to encrypt Kerberos tickets
* **Secret is stored ONLY on the Domain Controller**
  * Not on servers
  * Not in memory
  * Not in any local file
* Compromising the machine does **not** reveal the secret

***

## **Credential Guard Protection**

* Without CG → standard DC storage
* With CG → secret gets hardware-isolated protection

***

## **Secret Rotation**

* Auto-rotated periodically (like gMSA)
* **But unlike gMSA:**
  * gMSA: machines download the secret → can be stolen
  * dMSA: secret never leaves the DC → machine authenticates by proving identity, not by retrieving the secret

***

### The Migration Process Overview

The migration allows you to transition from an existing service account (the "superseded account") to a new dMSA without disrupting services. Think of it like switching from a manually-locked door to an automatic smart lock while keeping the door operational throughout the transition.

### Step 1: Initiating Migration

**Trigger Command:**

```
Start-ADServiceAccountMigration
```

**What happens behind the scenes:**

* This cmdlet calls an LDAP rootDSE operation named `migrateADServiceAccount`
* It requires three pieces of information:
  * The Distinguished Name (DN) of the new dMSA (where you're going)
  * The DN of the superseded account (where you're coming from)
  * A constant indicating you want to start the migration

### Key Attributes Tracking Migration State

**msDS-DelegatedMSAState** - This is the control panel that shows what stage the migration is in:

* **Value 0:** Unknown state (possibly disabled, but not officially confirmed)
* **Value 1:** Migration actively in progress
* **Value 2:** Migration successfully completed
* **Value 3:** This is a standalone dMSA (never migrated from anything)

### Additional Attributes Involved

**On the dMSA object itself:**

* `msDS-GroupMSAMembership`: A whitelist of who can use this dMSA
* `msDS-ManagedAccountPrecededByLink`: Points back to the old account being replaced

**On the superseded (old) account:**

* `msDS-SupersededManagedAccountLink`: Points forward to the new dMSA
* `msDS-SupersededServiceAccountState`: Mirrors the migration state values

These create a bidirectional link between old and new accounts, like leaving forwarding addresses when you move.

### Step 2: What Happens When Migration Starts

When you trigger `Start-ADServiceAccountMigration`, Active Directory makes these changes:

1. **Sets the state flag:** `msDS-DelegatedMSAState` becomes 1 (migration in progress)
2.  **Grants temporary permissions:** The dMSA's security descriptor is modified to give the old account:

    * Read permissions on the dMSA object
    * Write access specifically to the `msDS-GroupMSAMembership` attribute

    This allows the old account to inform the dMSA about which systems are still using it.
3. **Establishes the relationship links:**
   * dMSA's `msDS-ManagedAccountPrecededByLink` → points to old account
   * Old account's `msDS-SupersededManagedAccountLink` → points to dMSA
   * Old account's `msDS-SupersededServiceAccountState` → set to 1

**Current state:** The dMSA exists but isn't fully operational yet. It's in a learning phase, discovering which systems depend on the old account. Services continue using the old account normally.

### Step 3: Completing the Migration

**Completion Command:**

```
Complete-ADServiceAccountMigration
```

**What happens now:**

1. **Configuration inheritance:** The dMSA adopts critical settings from the superseded account:
   * **SPNs (Service Principal Names):** These are like the account's identification badges for Kerberos authentication
   * **Delegation settings:** Permissions for this account to act on behalf of others
   * **Other sensitive attributes:** Various security and operational configurations
2. **Old account retirement:** The superseded account is disabled (not deleted, but made inactive)
3. **State finalization:** Both `msDS-DelegatedMSAState` and `msDS-SupersededServiceAccountState` are set to 2 (completed)

**Result:** Any service that previously authenticated using the old account will now seamlessly use the dMSA instead. The transition is complete.

> Superseded = Replaced/Old/Outgoing

## Abusing the dMSA Migration Process for Privilege Escalation (Bad-successor)

After a normal migration, the KDC grants a dMSA **all the permissions of the superseded (old) account**.\
This is by design the purpose of migration is to let the new dMSA behave exactly like the old service account:

* Same SPNs
* Same delegation settings
* Same group memberships
* Same access rights

From a security perspective, this automatic privilege inheritance is extremely interesting.

During research, one specific behavior stood out:

#### **The KDC decides who the dMSA is replacing based on a single attribute:**

**`msDS-ManagedAccountPrecededByLink`**

Whatever account DN is stored in this attribute is considered the **superseded account**.\
So when the dMSA authenticates:

* KDC reads `msDS-ManagedAccountPrecededByLink`
* Builds the PAC using the permissions/SIDs of that account
* Gives the dMSA those same privileges

This is the foundation of the BadSuccessor attack chain.

***

## Thinking Like an Attacker

If AD copies permissions automatically based on one attribute, the next question is:

> Can an attacker influence this attribute to escalate privileges?

The researchers tested this idea with two different attack paths.

***

## **Privilege Escalation Technique 1: Gaining Access to Use the dMSA**

#### Goal

Become an authorized user of the dMSA so you can authenticate as it.

#### Why this works

During migration, AD temporarily grants the **superseded account**:

* **Read access** to the dMSA
* **Write access** to `msDS-GroupMSAMembership`\
  (the list of accounts allowed to use the dMSA)

If we can pretend to be the superseded account, we automatically get these rights.

#### Attack Steps

1. **Create a fake “superseded account”**
   * Put it in an OU you control
   * You now fully control this fake account
2. **Initiate a migration:**\
   FakeAccount → Target dMSA
   * This makes AD think the dMSA is replacing that fake account
3. **AD automatically updates permissions:**\
   The fake account now receives:
   * Read access to the dMSA
   * Write access to `msDS-GroupMSAMembership`
4. **Use the fake account to add your attacker account**

* &#x20;Add yourself into the dMSA’s authorized users list

1. **Now you can authenticate as the dMSA**\
   You have full usage access to the dMSA account.

This completes Attack #1.

***

## **Privilege Escalation Technique 2: Full Privilege Inheritance (BadSuccessor)**

Attack 1 gives you the ability to use the dMSA.

Attack 2 gives you **admin rights**.

#### Goal

Make the dMSA inherit privileges from a high-privilege account\
(e.g., Domain Admin)

#### Key Attribute

`msDS-ManagedAccountPrecededByLink`

When the dMSA authenticates, the KDC builds its PAC based on the account referenced here.

So if this attribute points to a Domain Admin:

* &#x20;The dMSA becomes a Domain Admin in the PAC
* Full domain compromise

#### Attack Steps

1. **Control the dMSA**\
   (either directly, or by completing Attack 1)
2. **Modify the PrecededByLink attribute:**

```
msDS-ManagedAccountPrecededByLink = <DN of Domain Admin / privileged account>
```

3. Set migration state to “completed”:

```
msDS-DelegatedMSAState = 2
```

1. **Authenticate as the dMSA**
   * The KDC now builds a PAC using the privileged account’s SIDs
   * You instantly gain all privileges of that high-value account

This completes Attack #2.

***

## Combining Both Attacks (Full BadSuccessor Chain)

If you start with only low-level rights:

#### Step 1 → Use Attack 1

Add yourself to the dMSA’s authorized users list

#### Step 2 → Use Attack 2

Make the dMSA inherit Domain Admin privileges

#### Step 3 → Authenticate as the dMSA

You now have **Domain Admin** rights\
without touching the real Domain Admin account\
and without triggering its logins.
