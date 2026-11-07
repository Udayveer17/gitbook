---
icon: box-isometric-tape
---

# Constrained Delegation

## Constrained Delegation

If you read my [previous post on Unconstrained Delegation](https://www.udayxd.xyz/notes/kerberos/unconstrained-delegation), you already know how the story starts. A service is trusted with a bit too much power, and that trust becomes the door an attacker walks through. Constrained Delegation is the more careful, more "locked down" version of that same idea. On paper it sounds safer. In practice, if it is set up wrong, it hands an attacker the keys to the domain just as easily.

I am writing this because understanding this topic properly took me a genuinely painful amount of time. I read the same paragraphs about S4U2Self and S4U2Proxy over and over and kept nodding along without actually getting it. So this post is written the way I wish someone had explained it to me the first time, slow, with a real story attached to it, instead of just diagrams and acronyms thrown at my face.

### What Constrained Delegation actually is

In plain words, Constrained Delegation lets a service act on behalf of a user, but only toward a specific, pre-approved list of other services. Unlike unconstrained delegation, where a service can impersonate a user anywhere in the domain, constrained delegation puts a leash on it. The service can only pretend to be that user when talking to services that are explicitly named on its allow list, stored in an attribute called `msDS-AllowedToDelegateTo`.

That sounds reasonable and safe at first glance. Locked down, controlled, only specific services allowed. What could possibly go wrong?

A lot, actually. The problem, as always with Active Directory, is in the details of how this trust is actually granted and verified. Keep that question in your back pocket as we go, because the answer to it is the entire attack.

### Scenario 1: The classic example, a login, a web server, and a file server

To really get this, let us walk through the everyday, non-attack version first. This is the situation this whole feature was built for in the first place.

<figure><img src="../.gitbook/assets/image (66).png" alt=""><figcaption></figcaption></figure>

Uday opens his browser and logs into WEB01. But here is the catch, he does not log in using Kerberos. Maybe it is a plain login form with a username and password, maybe it is some other authentication method entirely. The point is, from Kerberos's point of view, this login never happened. There is no Kerberos ticket anywhere that proves Uday authenticated to anything.

Now WEB01 needs to go fetch something from FILE01, but it needs to fetch it as Uday, not as itself. Access control on FILE01 depends on who is asking, so WEB01 cannot just show up as a generic service account. It needs a way to prove to FILE01 that Uday is the one asking.

So pause here for a second and think about it like WEB01 would. You have someone sitting in front of you who typed the right password into a form. Kerberos, the system everyone else in the domain trusts, has no idea this person exists. How do you bridge that gap?

This is exactly the gap that Protocol Transition was built to close. In short, Protocol Transition is the mechanism that lets a service take a non-Kerberos proof of identity, like that login form, and convert it into something Kerberos understands, a proper Kerberos ticket. It is literally transitioning from one kind of proof into another.

The actual mechanism that does this conversion is called S4U2Self. Here is the part that confused me the most, so I want to slow down here.

In S4U2Self, WEB01 requests a service ticket for itself, but asks the KDC (the Kerberos service running on the Domain Controller) to put Uday's identity information inside that ticket. In other words, WEB01 says, give me a ticket to myself, but let it represent Uday.

The obvious question I kept getting stuck on was, why would the KDC agree to that? Why would it just believe WEB01 when WEB01 claims that Uday logged in?

Sit with that question for a moment, because it is the whole point of this post. The answer is almost disappointing in how simple it is.

The KDC never actually checks whether Uday really logged in anywhere. It never asks Uday to prove anything. All it checks is one flag on WEB01's account in Active Directory, called `TRUSTED_TO_AUTH_FOR_DELEGATION`. This flag can only be set by an administrator, and it basically tells the KDC, this machine is trusted, whatever it claims about a user's identity, believe it.

If that flag is present, the KDC hands WEB01 a ticket. That ticket's service name is still WEB01 itself, but the identity data packed inside it, called the PAC, belongs to Uday. WEB01 now holds proof, at least as far as Kerberos is concerned, that Uday is the one it is dealing with.

<figure><img src="../.gitbook/assets/image (67).png" alt=""><figcaption><p>this one is for Unconstranied Dalegation but it look's same in constranied</p></figcaption></figure>

Now comes the second half, S4U2Proxy. WEB01 takes that ticket it just got and goes back to the KDC, this time asking for a ticket to FILE01, on Uday's behalf. The KDC does not just hand this over blindly. It checks WEB01's `msDS-AllowedToDelegateTo` list. If FILE01 is on that list, the KDC issues a ticket for FILE01, with Uday's identity baked into it. WEB01 can now walk up to FILE01 and access it as if it were Uday.

<figure><img src="../.gitbook/assets/image (68).png" alt=""><figcaption></figcaption></figure>

Here is where the real misconfiguration lives, and once this clicked for me, everything else made sense. At no point in this entire flow does the KDC verify that Uday genuinely logged into WEB01. It never checks with Uday. It only trusts the flag on WEB01's account. So if an attacker manages to control WEB01, or any account with that flag set, they do not need Uday at all. They can simply tell the KDC, give me a ticket claiming to be the Administrator, and the KDC will comply, because all it is checking is whether the requesting account is trusted to make that claim.

Read that last sentence again if you need to. This is the entire attack in one line.

Once the attacker has that Administrator-labeled ticket, they run it through S4U2Proxy exactly the same way. Whatever is sitting in WEB01's `msDS-AllowedToDelegateTo` list becomes fair game, except now it is being accessed as the Administrator, not as some low privilege user. If that list contains something like `cifs/dc.example.com`, the attacker walks away with a ticket that lets them access the Domain Controller's file services as the actual Administrator.

This example used a machine account, but the exact same weakness applies to a regular user account too, if that account has the delegation flag and an allowed list set on it.

### S4U2Self and S4U2Proxy, just the two of them

If the flow above made sense and you just want the two mechanisms laid out on their own, without the story around them, this section is for you. Skip it if you already have it, or read it as a recap before moving on.

#### S4U2Self

This is the part where a service asks the KDC for a ticket to itself, but requests that the ticket represent a specific user instead of the service.

A few things worth knowing about it:

* Any domain joined account can call S4U2Self by default. No special permission is needed to make the request itself.
* The KDC never verifies the claimed user actually authenticated anywhere because we have this flag `TRUSTED_TO_AUTH_FOR_DELEGATION`. It only checks whether the requesting account is allowed to make this kind of claim.
* That permission check comes down to a single flag on the requesting account, `TRUSTED_TO_AUTH_FOR_DELEGATION`, which only an administrator can set.
* The resulting ticket has the requesting service as the `sname`, but the PAC inside it, meaning the identity and group membership data, belongs to the impersonated user.
* If the flag is missing, the KDC still hands over a ticket, it does not refuse the request outright. The difference is the ticket comes back without the forwardable flag set, which quietly kills its usefulness for the next step.

In short, S4U2Self answers one question only, can this service claim to be someone else. It says nothing about where that claim can be used.

#### S4U2Proxy

This is the second half, where the service takes the ticket it got from S4U2Self and exchanges it for a ticket to a completely different, specific service, still on behalf of the same impersonated user.

A few things worth knowing about it:

* It requires a forwardable evidence ticket. If the S4U2Self ticket was not forwardable, this step is rejected immediately, no matter how convincing the claimed identity looks.
* The KDC checks the requesting service's `msDS-AllowedToDelegateTo` attribute. Only services listed there can be requested through this step.
* If the target service is not on that list, the request is denied, even if the identity claim itself was accepted without question.
* The resulting ticket has the target service as the `sname`, and the same impersonated user's data in the PAC.

In short, S4U2Proxy answers a different question, given that this service can claim to be someone else, where exactly is it allowed to take that claim.

Put them side by side and the split becomes obvious. S4U2Self decides who you can pretend to be. S4U2Proxy decides where you are allowed to walk in wearing that disguise. An attacker needs both pieces to line up, a flag that lets the lie be told, and a list that lets the lie go somewhere useful.

### Scenario 2: How this looked on a real box

This is not just a theory exercise. I ran into this exact setup while working through a HackTheBox machine called Painters, and seeing it on a real BloodHound graph is what finally made the concept stick for me. Nothing beats watching the theory turn into an actual command on your screen.

The graph showed three nodes. A compromised machine account, `PNT-SVRBPA$`, had two rights over a user called BLAKE, `ForceChangePassword` and `GenericWrite`. BLAKE, in turn, had an `AllowedToDelegate` edge pointing straight at the Domain Controller. That single edge meant BLAKE was already configured for constrained delegation, and its target was the Domain Controller itself.

<figure><img src="../.gitbook/assets/image (69).png" alt=""><figcaption></figcaption></figure>

Since I already had control over `PNT-SVRBPA$`, and it had the right to force a password change on BLAKE, I did not need to know BLAKE's original password at all. I just set a new one. From there, with BLAKE's fresh credentials in hand, and knowing BLAKE was trusted for delegation toward the Domain Controller, the rest of the chain fell into place.

One command was all it took:

```
getST.py -spn "cifs/DC.PAINTERS.HTB" -impersonate "Administrator" "PAINTERS.HTB"/"blake":"uday@1234"
```

Breaking that down using everything from above, this tells BLAKE to run S4U2Self and claim to be the Administrator, then immediately run S4U2Proxy for the `cifs` service on the Domain Controller. Because BLAKE was trusted for delegation, the KDC handed over the ticket without ever questioning whether the real Administrator had anything to do with this. And because `cifs/DC.PAINTERS.HTB` was sitting right there in BLAKE's allowed list, the request went through cleanly.

<figure><img src="../.gitbook/assets/image (70).png" alt=""><figcaption></figcaption></figure>

The end result was a valid ticket for the Administrator against the Domain Controller's `cifs` service. From there, it was a short hop to a DCSync attack, which meant full control of the domain.

<figure><img src="../.gitbook/assets/image (72).png" alt=""><figcaption></figcaption></figure>

### Why this matters

The thing about constrained delegation that makes it dangerous is not that it is a bug. It is working exactly as designed. The flaw is in how easily that design can be misused once an attacker gets a foothold on the wrong account. Two things need to line up for this attack to work, a delegation target on the allowed list, and the `TRUSTED_TO_AUTH_FOR_DELEGATION` flag enabling the identity claim in the first place. When both are present on an account you can compromise, you are effectively one command away from becoming anyone that account is allowed to impersonate.
