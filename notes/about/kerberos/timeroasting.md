---
icon: hourglass-start
---

# Timeroasting

## Timeroasting The Attack That Asks Nicely and Gets Your Hashes Anyway

I want to tell you about an attack that still kind of blows my mind every time I think about it. No credentials. No foothold. You just show up on the network, ask the Domain Controller a question it was never supposed to answer, and walk away with password hashes for every computer account in the domain.

It is called Timeroasting. And it works because of NTP.

***

### Wait, NTP? The Clock Sync Protocol?

Yeah. Bear with me.

Windows machines sync their clocks with the Domain Controller over NTP. Pretty standard stuff. The problem with vanilla NTP is that it has no authentication  a man-in-the-middle attacker could spoof the response and mess with the client's clock. Microsoft knew this, so they added a custom extension where the DC signs its NTP response using a MAC derived from the requesting computer's password hash.

The formula looks like this:

```
MAC = MD5( MD4(password) || NTP-response )
```

The idea being only the real DC knows the computer's password hash, so only the real DC can produce a valid signature. The client checks it, clock is synced, everyone is happy.

Except nobody thought about what happens when the _requester_ is not who they claim to be.

***

### The Part Where It Falls Apart

When a computer sends an NTP request, it includes its RID basically just a number that identifies its account in the domain. The DC looks up that RID, grabs the corresponding password hash, and signs the response.

Here is the thing though. **The DC never checks if you are actually that computer.**

You can send an NTP request with literally any RID you want. The DC will look it up and sign the response anyway. No authentication on your end. No questions asked. It just... does it.

So an attacker sitting outside the domain with zero credentials can loop through every possible RID, send a request for each one, and collect a signed response for all of them. What they end up with is a salted password hash for every single computer account in the domain. For free. Over a protocol that nobody monitors.
