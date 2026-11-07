---
icon: block-brick
cover: ../.gitbook/assets/Data-Center-Security-Services-by-SGS.jpg
coverY: 0
---

# Hardening Apache Redirectors

## Hardening Apache Redirectors for C2: A Practical Walkthrough

> **Disclaimer:** This post is written purely for educational purposes, It is not intended to be used against any system you do not have explicit, written permission to test. Everything here is for learning how defensive evasion and infrastructure hardening works, nothing more.

Before we get into hardening, you should already have a basic idea of what redirectors are and why they exist. If you have not read up on this yet, this post from me is a good primer: [Apache Redirectors Notes](https://www.udayxd.xyz/notes/redirectors/apache-redirectors). Once you have gone through that, come back here and we will build on top of it.

### Why Hide Your C2 in the First Place

So why should a red teamer even bother hiding their C2 server? Because there is a constant stream of threat intel folks and black hat types scanning the internet for exposed infrastructure. In a real engagement, if some threat intel analyst manages to take down your C2 mid operation, that is a pretty embarrassing look. Nobody wants to be the team that got caught because their infrastructure was sitting out in the open with zero protection.

### Redirect, Don't Block

When it comes to protecting Apache redirectors, the better strategy is to redirect traffic to a different location rather than blocking it outright or throwing a 403 Forbidden. There are a few solid reasons for this:

* **Avoiding detection:** If a scanner or defender gets hit with a 403 Forbidden, they can immediately tell their traffic got blocked, which tips them off that defensive measures are in place. Redirecting instead makes it much harder for them to tell their traffic was intercepted at all.
* **Providing misleading information:** Redirecting traffic somewhere else can actively mislead defenders or scanners. For example, sending them to a fake login page can distract them from ever finding the real one.
* **Maintaining categorization:** Web categorization tools crawl domains with bots to figure out what kind of content they host. We can use this to our advantage by pointing the bot toward something like a finance or healthcare page, so the domain keeps its "safe" categorization even after repeated rescans.

### Setting Up the First Rule: User-Agent Filtering

<figure><img src="../.gitbook/assets/image (58).png" alt=""><figcaption></figcaption></figure>

```
# Define Redirection Target based on Attacker's Domain Name
Define REDIR_TARGET https://www.microsoft.com/en-us
# This is where a normal, non-beacon user will get sent

# First Rule: User-Agent Check
RewriteCond %{HTTP_USER_AGENT} (google|yandex|bingbot|Googlebot|bot|spider|simple|BBBike|wget|cloudfront|curl|Python|Wget|crawl|baidu|Lynx|xforce|HTTrack|Slackbot|netcraft|NetcraftSurveyAgent|Netcraft) [NC]
RewriteRule ^(.*)$ ${REDIR_TARGET}
```

Once this rule is in place, restart Apache to apply the changes:

```
apache2 restart
```

The `[NC]` flag stands for "No Case." It means the user agent match is case insensitive, so it will catch variations regardless of capitalization.

After restarting, test the redirector with curl:

```
curl https://domainname
```

<figure><img src="../.gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

Example output:

```
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="https://www.microsoft.com/en-us">here</a>.</p>
</body></html>
```

### Confirming Legitimate Traffic Still Gets Through

We also want to confirm that traffic with a legitimate browser user agent actually makes it through to the team server, instead of getting caught by our own filter.

<figure><img src="../.gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

```
curl -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:147.0) Gecko/20100101 Firefox/147.0" https://domainname/
```

### Adding a Custom Header Check

Next, let's add a second layer using a custom header. Here is the rule:

```
# Second Rule: Custom Header Check
RewriteCond %{HTTP:DNT} ^$
RewriteRule ^(.*)$ https://www.microsoft.com/en-us [L,R=301]
```

Here's how this works: Apache checks the first rule, and if the request matches it, that request gets handled there and Apache stops processing further rules. If it does not match the first rule, Apache moves on to check the second one. In this case, we are checking whether the request is missing a specific custom header (in this case, DNT). If the header is missing, the request gets redirected away.

Now we need to make our beacon traffic actually send this header, so it passes the check. This means updating our C2 profile to include the custom header in Get and post both.

<figure><img src="../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

After updating the profile, restart Apache and test again with curl. Without the header, the request should get redirected away (the connection to the real target fails). With the header added, it goes through:

```
curl --header "DNT: 1" -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:147.0) Gecko/20100101 Firefox/147.0" https://domain.com/dntheader
```

### Comms Filtering: Cobalt Strike Profile URIs

At this point we have decent protection against automated scans, but a sharp analyst could still figure out the user agent and header just by inspecting traffic closely. So the last layer we will add is filtering based on the exact URIs configured in the team server's Malleable C2 profile.

In your Cobalt Strike team server profile, the GET and POST requests use two distinct URIs:

```
http-post {

    set uri "/uday/xd.html";    # URI used for POST block.
    set verb "POST";                 # HTTP verb used in POST block. Can be GET or POST

}

http-get {

    set uri "/lamo/xd.shtml";    # URI used for GET requests
    set verb "GET";

}
```

We can check the redirector for these exact URIs, and if a request does not use them, we send it off to our decoy host instead. Here is the full config with this added:

```
# Define Redir target
Define REDIR_TARGET https://www.microsoft.com/en-us
Define C2_Server 10.10.10.204:4443

# Define the paths for the CS GET and POST requests
Define CS_GET /lamo/xd.shtml
Define CS_POST /uday/xd.html

# First Rule: User-Agent Check
RewriteCond %{HTTP_USER_AGENT} (google|yandex|bingbot|Googlebot|bot|spider|simple|BBBike|wget|cloudfront|curl|Python|Wget|crawl|baidu|Lynx|xforce|HTTrack|Slackbot|netcraft|NetcraftSurveyAgent|Netcraft) [NC]
RewriteRule ^(.*)$ ${REDIR_TARGET}

# Second Rule: Custom Header Check
RewriteCond %{HTTP:DNT} ^$
RewriteRule ^(.*)$ https://www.microsoft.com/en-us [L,R=301]

# Third Rule: URI Check for GET Request
RewriteCond %{REQUEST_URI} ^${CS_GET}.*$
RewriteRule ^${CS_GET}.*$ %{REQUEST_SCHEME}://${C2_Server}%{REQUEST_URI} [P]
ProxyPassReverse / %{REQUEST_SCHEME}://${C2_Server}

# Fourth Rule: URI Check for POST Request
RewriteCond %{REQUEST_URI} ^${CS_POST}.*$
RewriteRule ^${CS_POST}.*$ %{REQUEST_SCHEME}://${C2_Server}%{REQUEST_URI} [P]
ProxyPassReverse / %{REQUEST_SCHEME}://${C2_Server}

# Remove generic proxy pass rules!
```

At this point, the redirector is filtering traffic based on four things:

* User-Agent
* DNT header
* GET URI
* POST URI

Only requests that match every single one of these get forwarded to the team server. Everything else gets bounced.

To test this, first send a request without the correct URI:

```
curl --header "DNT: 1" -A "Mozilla/5.0" https://domain.com/
```

<figure><img src="../.gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>

The response should be a plain 404:

```
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html>
<head>
<title>404 Not Found</title>
</head>
<body>
<h1>Not Found</h1>
<p>The requested URL was not found on this server.</p>
</body>
</html>
```

This is expected. Since we removed the generic ProxyPassReverse rules, only requests matching the configured GET or POST URI get forwarded at all. Everything else just hits a dead end.

Now let's confirm it actually works when we use the correct URI, so the team server responds like it would to a real beacon:

```
curl --header "DNT: 1" \
-A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:147.0) Gecko/20100101 Firefox/147.0" \
https://domain.com/css3/index2.shtml
```

If everything is set up right, this request gets forwarded straight to the Cobalt Strike team server and comes back with the configured profile response instead of a 404.

### Wrapping Up

At this point, the redirector is reasonably hardened. This is by no means the ceiling though, there is a lot more you can layer on top depending on how creative you want to get: geo-fencing, ASN blocking, rate limiting, TLS fingerprint checks, and so on. The four checks covered here (user agent, custom header, GET URI, POST URI) are a solid baseline to build from.
