---
icon: truck-moving
---

# Resource-Based Constrained Delegation (RBCD)

## Resource-Based Constrained Delegation

If you haven't read my earlier post on [Constrained Delegation](https://www.udayxd.xyz/notes/kerberos/constrained-delegation), go read that first, this one builds straight on top of it. S4U2Self, S4U2Proxy, the PAC, all of that comes back here. RBCD isn't some brand new mechanism you gotta learn from scratch, it's Resource-Based Constrained Delegation: When The Target Decides Who To Trust

Okay so if you read my [Constrained Delegation post](https://www.udayxd.xyz/notes/kerberos/constrained-delegation), you already have basically everything you need for this one. Same S4U2Self, same S4U2Proxy, same PAC stuff. RBCD isn't some new thing you gotta relearn from zero, it's the exact same two steps, just the trust part sits on a different account this time.

If you already got S4U2Self and S4U2Proxy down, this whole post is gonna click way faster than you expect.

#### quick recap first

Last time we looked at two types of delegation and both had the same setup underneath. The permission always sat on the source, meaning whoever was doing the impersonating.

Unconstrained had `TRUSTED_FOR_DELEGATION` on the source, and once that's on, it can grab a full copy of whoever's TGT shows up, no limits on where it takes that identity after.

Constrained had a list on the source too, `msDS-AllowedToDelegateTo`, just way more specific about which services it could impersonate people toward.

Both of these needed some Domain Admin level thing to even set up in the first place, `SeEnableDelegationPrivilege` for one, admin rights to touch that list for the other. So realistically you needed a solid foothold already, or you got lucky and found some old box that already had this misconfigured.

RBCD flips that one part. Instead of the source deciding, the target decides.

#### the actual idea

So instead of the source holding a list of where it's allowed to go, now the target holds a list of who's allowed to come talk to it.

The attribute is `msDS-AllowedToActOnBehalfOfOtherIdentity` and it lives on the target, not the source. So the target is basically saying, these accounts right here, I trust them, they can act on behalf of other users when they hit me up.

And this changes a lot more than it seems like at first. You don't need Domain Admin to edit this attribute. All you need is something like `GenericWrite`, `GenericAll`, or `WriteDacl` on the target computer object. And these show up more than people think, some ACL got messed up somewhere, some helpdesk group got handed more access than it should've, or nobody ever went back and cleaned it up.

This is why RBCD ends up being one of the first things worth checking in most AD environments. You don't need a domain admin foothold. You just need write access on one machine object, anywhere.

#### how the attack actually goes

Say you've got write access on a target machine, let's just call it FILE01.

**step 1, get yourself an account with an SPN**

You need an account that has a Service Principal Name on it, something that looks like a service or machine account, since that's what the S4U stuff expects. If you already control one, use it. If not, machine account quota still works here, most users can make up to ten computer accounts.

```
bloodyAD --host DC1.domain.local -d domain.local -u youruser -p yourpass add computer attack Password123@
```

**step 2, get your account on the target's trust list**

Here's the part that's actually RBCD. You edit FILE01's `msDS-AllowedToActOnBehalfOfOtherIdentity` and put your account in there.

with bloodyAD:

```
bloodyAD --host DC1.domain.local -d domain.local -u youruser -p yourpass set rbcd FILE01$ attack$
```

you can also do this with `rbcd.py` from Impacket, which lets you read, write, or clear it separately, which is honestly nice if you wanna check what's already sitting there first before you overwrite anything.

```
# read the current value
rbcd.py -delegate-to "$TARGET$" -dc-ip "$DC_HOST" -action 'read' "$DOMAIN/$USER:$PASSWORD"

# add your account to msDS-AllowedToActOnBehalfOfOtherIdentity
rbcd.py -delegate-from "$CONTROLLED_ACCOUNT" -delegate-to "$TARGET$" -dc-ip "$DC_HOST" -action 'write' "$DOMAIN/$USER:$PASSWORD"
```

the account doesn't need to be one you made yourself either, any account whose creds you know works, as long as it's got an SPN on it for the normal version of this. there's also an SPN-less version but that's its own thing to go read about separately.

one more way people get this attribute written, comes up a lot with relays. if you're relaying NTLM auth with `ntlmrelayx`, there's a `--delegate-access` flag, and instead of just relaying somewhere, it uses that session to write your account straight into the target's delegation list. same result, just a different way in.

**step 3, run S4U**

now you use your controlled account's creds and go straight for the S4U2Self plus S4U2Proxy chain, impersonating whoever, usually Administrator.

```
getST.py -spn 'cifs/target' -impersonate "Administrator" -dc-ip "$DC_IP" "$DOMAIN"/"$ACCOUNT_WITH_SPN":"$PASSWORD"
```

okay this part messed with my head the most the first time, so slowing down here again.

your controlled account almost certainly doesn't have `TRUSTED_TO_AUTH_FOR_DELEGATION` on it. in regular constrained delegation that flag was required, because without it the S4U2Self ticket comes back non forwardable and S4U2Proxy just throws it out. RBCD just skips that whole check. when the target has that `msDS-AllowedToActOnBehalfOfOtherIdentity` list, the KDC checks that list during S4U2Proxy instead of caring about your forwardable flag at all. your account's on the list, KDC hands you a valid ticket for the target, PAC still says Administrator, no flag needed on your side.

so really the whole thing is, S4U2Self gets you a ticket claiming to be Administrator tied to your own account. S4U2Proxy doesn't even look at your permissions at that point, it looks at the target's own list, and since you just put yourself on that list in step 2, it just goes through.

**step 4, use it**

load it up and hit the target.

```
export KRB5CCNAME=Administrator@cifs_FILE01@DOMAIN.LOCAL.ccache
```

depending on what service you asked for, this could be file share access, a way to dump creds, or if the target's the DC, straight into a DCSync.

#### why this actually matters

the reason RBCD shows up so much in real environments is that the bar dropped from "you need Domain Admin to even touch this" to "one write permission on one machine object gets you the whole chain." a single `GenericWrite` that slipped in through some nested group, some old delegation nobody cleaned up, or a helpdesk permission scoped too wide, and you're already through, no privileged account needed first.

it's also why this works so well with relays. coerce or relay auth from something that already has write access to a target, and you skip needing a compromised user at all, straight from network position to writing the attribute to pulling the ticket.

#### all three side by side

worth just stepping back for a second. unconstrained trusts the source completely, no limits, only an admin can set it up. constrained trusts the source too but only for specific services, still admin only. RBCD moves that trust to the target, and now anyone with basic write access on that one object can open the door themselves.

once you see it that way, these stop being three separate attacks to memorize and just become the same S4U2Self and S4U2Proxy flow, wearing three different permission setups.

Ngl if you've already got S4U2Self and S4U2Proxy sitting comfortably in your head, this one's gonna feel short. And honestly that's kind of the point. Once the core two step exchange makes sense, RBCD is really just one small but sneaky change in who gets to decide the trust.
