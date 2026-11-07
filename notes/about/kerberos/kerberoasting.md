---
icon: dog
---

# Kerberoasting

## Kerberoasting (The Complete Breakdown)

### What's This All About?

So, Kerberoasting. If you've been in the infosec world for a bit, you've probably heard this term thrown around. It's one of those attack techniques that's been absolutely huge since Tim Medin first presented it back in 2014 at SANS HackFest. And honestly? It's still going strong today because it's just so damn effective.

Here's the basic idea: Kerberoasting lets you steal encrypted password hashes for service accounts in Active Directory environments, then crack them offline at your own pace. No need for admin rights, no malware required, and best of all no account lockout risk. It's basically a pentester's dream technique.

According to CISA (the Cybersecurity Infrastructure and Security Agency), Kerberoasting is one of the most time-efficient ways to elevate privileges and move laterally through a network. That's a pretty big endorsement coming from the US government, right?

### A Quick Kerberos Refresher

Before we dive deep into Kerberoasting, let's make sure we're all on the same page about how Kerberos actually works. I promise to keep this brief.

<figure><img src="../.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

Here's the normal flow:

1. **User requests a TGT**: You (or your computer) send a request to the Key Distribution Center (KDC) - which in Windows domains is just the domain controller - saying "Hey, I'm Bob, give me a ticket-granting-ticket."
2. **You prove who you are**: Along with that request, you send an authenticator encrypted with your password to prove you're actually Bob.
3. **KDC sends back a TGT**: If everything checks out, the domain controller responds with a KRB\_AS\_REP that contains your TGT (encrypted with the DC's secret key) and a session key (encrypted with YOUR password).
4. **Request service tickets**: Now you can use that TGT to request service tickets (TGS) for specific resources on the domain - like a file share, SQL server, whatever.

Here's where it gets interesting for us as attackers.

### How Kerberos Uses SPNs

The Windows implementation of Kerberos uses something called Service Principal Names (SPNs) to figure out which account's hash to use when encrypting service tickets. Think of SPNs as unique identifiers that link services to specific accounts.

There are two main types:

**Host-based SPNs**: These are automatically created for computer accounts. For example, when you create a new computer called WINDOWS1 in the domain, it automatically gets SPNs like `HOST/WINDOWS1` and `HOST/WINDOWS1.testlab.local`. These are linked to the machine account (WINDOWS1$).

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

**Arbitrary SPNs**: These are manually registered SPNs, usually tied to user accounts. For example, a service account managing several MSSQL instances might have SPNs like `MSSQLSvc/SQL.domain.com:1433` registered in the user's servicePrincipalName attribute.

Now here's the key difference: Machine account passwords are randomly generated, super complex, and rotate every 30 days automatically. Good luck cracking those. But user account passwords? Those are set by humans, which means they're often weak, simple, and changed infrequently (if ever).

And that's exactly what Kerberoasting exploits.

### The Attack Explained

Alright, so how does this attack actually work? Let me break it down step by step.

#### Step 1: Find Vulnerable Accounts

First, you need to identify user accounts that have SPNs registered. You can do this with a simple LDAP filter:

```
&(objectCategory=person)(objectClass=user)(servicePrincipalName=*)
```

Here's a quick PowerShell script to enumerate these accounts:

```powershell
$search = New-Object DirectoryServices.DirectorySearcher([ADSI]"")
$search.filter = "(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*))"
$results = $search.Findall()
foreach($result in $results)
{
    $userEntry = $result.GetDirectoryEntry()
    Write-host "User : " $userEntry.name "(" $userEntry.distinguishedName ")"
    Write-host "SPNs"        
    foreach($SPN in $userEntry.servicePrincipalName)
    {
        $SPN       
    }
    Write-host ""
}
```

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

#### Step 2: Request Service Tickets

Here's the beautiful part: ANY domain user can request a service ticket for ANY SPN in the domain. You don't need elevated privileges. You don't need to be an admin. Just a regular domain user account.

When you request a service ticket for an SPN, the domain controller:

* Checks that your TGT is valid
* Looks up which account has that SPN registered
* Encrypts part of the service ticket with the hash of that account's password
* Sends the ticket back to you

The domain controller doesn't check if you're actually authorized to access that service - that's the service's job when you present the ticket. But we don't care about actually using the service. We just want that encrypted blob.

#### Step 3: Extract and Crack Offline

Once you have the service ticket, you extract the encrypted part and start cracking it offline. Since it's encrypted with the service account's password hash, if you can crack it, you recover the plaintext password.

The best part? You can take your time. No account lockout. No alerts from failed login attempts. Just you, your GPU, and a wordlist.

### Why This Attack Is So Prevalent

According to researchers at Rapid7 and CrowdStrike, Kerberoasting is super prevalent for a few key reasons:

**Low barrier to entry**: You only need one compromised user account - any user account - to pull this off. According to CISA, Russian state-sponsored APT actors love this technique because it's so accessible.

**Hard to detect**: Because you're using legitimate Kerberos functionality, traditional security tools often miss it. You're not exploiting a vulnerability or using malware. You're just requesting tickets like any normal user would.

**High success rate**: Service accounts tend to have weak passwords that rarely change. I've personally seen service accounts with passwords like "Summer2015!" that haven't been touched in years.

**No malware needed**: As CrowdStrike points out, traditional antivirus solutions are completely ineffective against Kerberoasting because there's no malicious code involved.

### The Evolution of Kerberoasting Techniques

The attack has evolved quite a bit since Tim Medin's original presentation. Let me walk you through the different approaches.

#### The "Old School" Method

Tim's original toolkit combined several tools:

1. Enumerate accounts with SPNs (using GetUserSPNs.ps1 or similar)
2. Request TGS tickets using `setspn.exe` or PowerShell
3. Extract tickets from memory using Mimikatz
4. Convert to crackable format with `tgsrepcrack.py` or `kirbi2john.py`
5. Crack offline with John the Ripper

This worked, but it was kinda clunky and required multiple tools.

#### The PowerShell Revolution

Around 2016, things got way easier thanks to some clever folks. As Harmj0y documented, Matan Hart (@machosec) realized that the .NET `KerberosRequestorSecurityToken` class had a `GetRequest()` method that returns the raw bytes of the Kerberos ticket.

This meant you could:

* Request tickets using pure PowerShell
* Extract the encrypted part without Mimikatz
* Output directly in John or Hashcat format

Harmj0y rolled this into a single function called `Invoke-Kerberoast` that handles everything:

```powershell
Invoke-Kerberoast -OutputFormat Hashcat | Format-List
```

<figure><img src="../.gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

It even works across domain trusts! And you can use the `-AdminCount` flag to only target highly privileged accounts.

#### Modern Tools

Today we have several excellent options:

**Rubeus** (by @harmj0y and team): A C# toolkit that started as a port of Kekeo and has become the gold standard for Windows-based Kerberoasting.

```
Rubeus.exe kerberoast
```

**Impacket's GetUserSPNs.py**: Perfect for Linux environments or when you want to attack from outside the domain.

```
GetUserSPNs.py domain.com/username:password -dc-ip 10.10.10.1
```

**NetExec**: Modern and actively maintained.

```
nxc ldap 10.10.10.1 -u user -p pass --kerberoast output.txt
```

### The Encryption Problem (And Why RC4 Matters)

Here's something that took the community a while to fully understand, and Harmj0y did an excellent deep dive on this in his "Kerberoasting Revisited" post.

When you request a service ticket, what encryption type you get back isn't just about what you request - it's about a property called `msDS-SupportedEncryptionTypes` on the service account.

#### Understanding msDS-SupportedEncryptionTypes

This is a 32-bit field that tells the KDC what encryption types an account supports:

* RC4\_HMAC\_MD5 (value: 4)
* AES128\_CTS\_HMAC\_SHA1\_96 (value: 8)
* AES256\_CTS\_HMAC\_SHA1\_96 (value: 16)

Here's the kicker: **Computer accounts have this property set by default to 28 (RC4 + AES128 + AES256), but user accounts don't have it set at all.**

When this property is undefined or set to 0, the default behavior is RC4 encryption. That's why Kerberoasting "just works" so well most service accounts return RC4 encrypted tickets by default, which are thousands of times faster to crack than AES.

<figure><img src="../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

Even weirder? Harmj0y discovered that even if you enable AES encryption on a user account, if you specifically request RC4, you'll still get an RC4 ticket. The only way to truly block RC4 is to disable it at the domain controller level, which might break a lot of stuff.

#### Cracking the Hashes

Once you have your tickets, cracking is straightforward.

**John the Ripper**:

```bash
john --wordlist=rockyou.txt --format=krb5tgs hashes.txt
```

**Hashcat** (way faster with GPUs):

```bash
hashcat -m 13100 hashes.txt wordlist.txt
```

The hash mode depends on encryption:

* RC4 (type 23): mode 13100
* AES256 (type 18): mode 19700 (much slower)

According to Microsoft's security blog, the use of GPUs has made password cracking significantly faster, which is why they're pushing so hard for organizations to move away from RC4 and use strong passwords.

### Real-World Attack Scenario

Let's say you're on a pentest and you've compromised a single domain user account - maybe through phishing or finding credentials in a file share. Here's how you'd leverage Kerberoasting:

1. **Enumerate**: Find all user accounts with SPNs

```powershell
Get-DomainUser -SPN | Select samaccountname,serviceprincipalname
```

2. **Prioritize**: Look for high-value targets - accounts with AdminCount=1 or SPNs suggesting important services (SQL, IIS, etc.)
3. **Roast**: Request tickets for these accounts

```powershell
Invoke-Kerberoast -OutputFormat Hashcat | Out-File hashes.txt
```

4. **Crack**: Fire up Hashcat with a good wordlist

```bash
hashcat -m 13100 hashes.txt rockyou.txt --rules rules/best64.rule
```

5. **Profit**: Use recovered credentials to access the service or pivot further

The beauty is you can do all this without triggering account lockouts or generating obvious failed login attempts.

### Advanced Techniques

#### The /tgtdeleg Trick

Harmj0y implemented something really clever in Rubeus based on a technique from @gentilkiwi's Kekeo toolkit.

The problem: The normal `KerberosRequestorSecurityToken` approach requests the highest supported encryption. So if a service account has AES enabled, you get an AES ticket (much harder to crack).

The solution: Use the Kerberos GSS-API to request a "fake" delegation ticket, which gives you a usable TGT for the current user. Then use that TGT to manually craft TGS requests where you specify you only support RC4.

```
Rubeus.exe kerberoast /tgtdeleg
```

<figure><img src="../.gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

Even better, the `/rc4opsec` flag filters out AES-enabled accounts and only targets accounts where RC4 is the highest supported encryption. This avoids the "encryption downgrade" detection that some defensive tools look for.

#### Cross-Domain Roasting

Kerberoasting works across domain trusts too! If you have access in one domain, you can often Kerberoast accounts in trusted domains.

```powershell
Invoke-Kerberoast -Domain external.domain.com
```

Harmj0y noted that inter-domain trust tickets actually use RC4 by default in most environments because the `msDS-SupportedEncryptionTypes` property on trusted domain objects is usually undefined.

### Detection and Defense

Alright, let's flip to the blue team perspective for a minute.

#### How to Detect Kerberoasting

**Monitor for unusual ticket requests**: According to Microsoft and CrowdStrike, you should watch for:

* Single users requesting multiple service tickets in a short timeframe
* Requests for tickets with RC4 encryption when AES should be used
* Ticket requests for accounts that aren't normally accessed

**Enable advanced auditing**: Microsoft Defender XDR can raise alerts (external ID 2410) for suspected Kerberos SPN exposure.

**Analyze Kerberos events**: Sean Metcalf has written extensively about this. Look at Event ID 4769 (service ticket requested) for anomalies, though be prepared for false positives.

**Use UEBA tools**: Solutions like Rapid7's Incident Command or CrowdStrike Falcon can baseline normal user behavior and flag anomalous Kerberos activity.

#### How to Prevent Kerberoasting

**Use Group Managed Service Accounts (gMSA)**: These are a game-changer. Microsoft introduced gMSAs specifically to address this problem. They automatically:

* Generate 120-character random passwords
* Rotate passwords automatically
* Handle SPN registration properly

Creating one is easy:

```powershell
New-ADServiceAccount sql-service
Install-ADServiceAccount sql-service
```

Microsoft also introduced Delegated Managed Service Accounts (dMSA) in Server 2025, which offer even better protection with Credential Guard integration.

**If you can't use gMSAs**: At minimum, ensure service accounts have:

* 25+ character randomly generated passwords (Microsoft recommends at least 14, but longer is better)
* AES encryption enabled
* Passwords audited regularly

**Audit your SPNs**: Regularly check which user accounts have SPNs and remove any that aren't needed. Every SPN on a user account is a potential attack vector.

**Disable RC4**: In a future update to Windows 11 24H2 and Windows Server 2025, Microsoft plans to disable RC4 by default. You can do this manually now, but test extensively first as it might break legacy applications.

**Monitor privileged accounts**: Accounts with AdminCount=1 or domain admin rights should get extra scrutiny. These are the juiciest Kerberoasting targets.

#### Responding to an Attack

If you detect an active Kerberoasting attack:

1. **Don't panic**: Take a breath. The attacker still needs to crack the passwords.
2. **Reset compromised credentials**: Change passwords for any potentially affected service accounts immediately.
3. **Enable MFA**: Multi-factor authentication makes stolen credentials much less useful.
4. **Implement least privilege**: Ensure service accounts only have the minimum necessary permissions.
5. **Consider incident response help**: Rapid7 and CrowdStrike both recommend engaging with detection and response vendors for faster remediation.
6. **Hunt for persistence**: If credentials were compromised, look for backdoors or additional compromised accounts.

### Bonus Section: Targeted Kerberoasting

Alright, here's a bonus technique for you guys that Harmj0y demonstrated back in 2017.

Let's say you've compromised an account that has `GenericWrite` or `GenericAll` rights over another user object in Active Directory. You want to leverage that victim user's access, but you don't want to reset their password (too obvious - they'll notice).

Here's the sneaky alternative:

**The Attack**:

1. Use your write permissions to add a fake SPN to the victim's account
2. Kerberoast that fake SPN to get their password hash
3. Crack it offline
4. Remove the fake SPN to cover your tracks

<figure><img src="../.gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

Everything you need is already in PowerView:

```powershell
# Check current SPNs
Get-DomainUser victimuser | Select serviceprincipalname

# Add fake SPN
Set-DomainObject -Identity victimuser -SET @{serviceprincipalname='nonexistent/BLAHBLAH'}

# Kerberoast it
$User = Get-DomainUser victimuser 
$User | Get-DomainSPNTicket | fl

# Clean up
Set-DomainObject -Identity victimuser -Clear serviceprincipalname
```

This is way stealthier than a password reset. The victim never knows anything happened. Obviously you still need a crackable password, but it's a great option when you have those specific DACL rights.

The downside? While the SPN modification itself might not be noticed immediately, there are event logs that can detect this if defensive teams are watching for suspicious attribute modifications. But in practice, most environments aren't monitoring for this.

### Final Thoughts

Kerberoasting has been around for over a decade now, and it's still incredibly effective. Why? Because it exploits fundamental design decisions in how Active Directory and Kerberos work together, not some patchable vulnerability.

The main takeaways:

* **For red teamers**: This should be in your standard playbook. It works, it's reliable, and tools like Rubeus make it trivial to execute.
* **For blue teamers**: Use gMSAs wherever possible. Seriously. And if you can't, at least ensure strong passwords and AES encryption for service accounts.
* **For everyone**: The fact that any domain user can request service tickets for any SPN is by design, not a bug. Defending against this requires good password hygeine, proper monitoring, and defense in depth.

As Microsoft points out in their security guidance, this is a team effort. No single control will completely prevent Kerberoasting, but layered defenses - strong passwords, gMSAs, AES encryption, monitoring, and least privilege - can make it much harder for attackers to succeed.

Stay safe out there, and remember: if a password was created by a human, it can probably be cracked by a determined attacker with enough time and GPU power.

***

### References

1. https://en.hackndo.com/kerberoasting/
2. https://www.rapid7.com/fundamentals/kerberoasting-attack/
3. https://www.microsoft.com/en-us/security/blog/2024/10/11/microsofts-guidance-to-help-mitigate-kerberoasting/
4. https://www.crowdstrike.com/en-gb/cybersecurity-101/cyberattacks/kerberoasting/
5. https://blog.harmj0y.net/powershell/kerberoasting-without-mimikatz/
6. https://blog.harmj0y.net/activedirectory/targeted-kerberoasting/
7. https://specterops.io/blog/2019/02/20/kerberoasting-revisited/
