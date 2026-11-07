---
icon: helicopter
---

# Apache redirectors

## Setting Up Apache Redirectors for Red Team Infrastructure

_This post is for educational purposes only. Everything covered here should only be used on infrastructure you own, in support of engagements you are explicitly authorized to perform. Misusing this information against systems you don't have permission to test is illegal._

### Prerequisites

Before you start, you'll need:

* **2 VPS servers** one will act as your redirector (the public-facing box), the other as your team server (kept hidden behind the redirector).
* **A domain name** something that isn't going to draw attention or get flagged by reputation services on day one.

That's it for the starting point. Everything else we'll build up from scratch.

### Step 1: Point Your Domain at Your VPS

First, we need DNS pointed at our redirector VPS.

1. Log into Cloudflare and select your domain.
2. Add an **A record** pointing to your VPS's public IP.
3. Make sure the proxy status is set to **DNS only** (the grey cloud, not orange) we don't want Cloudflare's proxy in front of this yet.

<figure><img src="../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>

Give it a couple of minutes to propagate, then check it from your terminal:

```bash
dig domainname.com +short
```

This should return your VPS's IP address. If it doesn't yet, wait another minute or two and try again propagation is usually quick but not always instant.

<figure><img src="../.gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>

Once that resolves correctly, your domain is live and pointed where it needs to be.

### Step 2: Install and Harden Apache

Now we set up the actual web server. Below is a setup script that installs Apache, layers on some basic hardening, and sets things up so the server doesn't immediately give away what it's running.

```bash
#!/bin/bash
# ============================================================================
# Apache Server Setup & Security Hardening Script
# ----------------------------------------------------------------------------
# Description:
# This script automates the installation and configuration of an Apache web
# server on an Ubuntu system. It includes security enhancements, SSL setup,
# and configurations to hide server identity.
#
# Features:
# - Updates system package lists
# - Installs and configures Apache2
# - Sets up SSL using Certbot
# - Enhances security with mod_security2
# - Hides Apache version and identity
# - Enables necessary modules for proxying and security
#
# Usage:
#   sudo bash server-setup.sh
# ============================================================================
set -e  # Exit immediately if a command exits with a non-zero status

# Update package lists
echo "Updating system packages..."
sudo apt update -y

# Install Apache
echo "Installing Apache web server..."
sudo apt install apache2 -y
sudo systemctl enable apache2

# Install Certbot for SSL certificate management
echo "Installing Certbot for SSL..."
sudo apt install certbot python3-certbot-apache -y

# Install and enable ModSecurity for Apache
echo "Installing ModSecurity for enhanced security..."
sudo apt install libapache2-mod-security2 -y
sudo a2enmod security2

# Disable default Apache site configuration
echo "Disabling default Apache site..."
sudo a2dissite 000-default.conf

# Enable essential Apache modules
echo "Enabling necessary Apache modules..."
sudo a2enmod proxy proxy_ajp proxy_http rewrite deflate headers \
    proxy_balancer proxy_connect proxy_html

# Disable directory listing
echo "Disabling directory listing..."
sudo a2dismod autoindex -f

# Secure Apache configuration
SECURITY_CONF="/etc/apache2/conf-available/security.conf"
echo "Applying security configurations..."
sudo sed -i "s/ServerSignature On/ServerSignature Off/g" "$SECURITY_CONF"
sudo sed -i "s/ServerTokens OS/ServerTokens Full/g" "$SECURITY_CONF"

# Obfuscate the server signature
echo "Obfuscating Apache server identity..."
echo "SecServerSignature Microsoft-IIS/10.0" | sudo tee -a "$SECURITY_CONF" > /dev/null

# Restart Apache to apply changes
echo "Restarting Apache to apply changes..."
sudo systemctl restart apache2

echo "Apache server setup and security hardening completed successfully!"
```

Run it with:

```bash
sudo bash server-setup.sh
```

A few things worth calling out about what this does:

* **`ServerSignature Off`** and the **`ServerTokens`** tweak stop Apache from advertising its exact version in error pages and response headers.
* **ModSecurity** gets installed as a baseline web application firewall layer.
* The server signature gets rewritten to claim it's IIS a small bit of misdirection so casual scanning doesn't immediately fingerprint it as Apache.
* Directory listing is disabled so nobody can browse your web root and see what's sitting there.

### Step 3: Set Up the Virtual Host and TLS Certificate

With Apache installed, we need a virtual host for our domain and a real, trusted TLS certificate not a self-signed one, since that alone can be a red flag.

```bash
#!/bin/bash
# ============================================================================
# Apache Virtual Host & SSL Setup Script
# ----------------------------------------------------------------------------
# Description:
# This script automates the setup of a virtual host for a given domain on an
# Apache web server. It also configures SSL using Certbot and ensures proper
# permissions for the website directory.
#
# Features:
# - Creates necessary directories for the domain
# - Sets up an Apache Virtual Host configuration
# - Enables the site and restarts Apache
# - Checks if the domain resolves via DNS
# - Acquires and installs an SSL certificate using Certbot
#
# Usage:
#   sudo bash domain-setup.sh example.com
# ============================================================================
set -e  # Exit on errors

# Validate input
if [[ -z "$1" ]]; then
    echo "Error: No domain provided. Usage: sudo bash $0 yourdomain.com"
    exit 1
fi

# Set important variables
MYDOMAIN=$1
VHOST_CONF="/etc/apache2/sites-available/${MYDOMAIN}.conf"
WEB_ROOT="/var/www/${MYDOMAIN}"
LOG_DIR="${WEB_ROOT}/logs"

echo "Configuring domain: $MYDOMAIN"

# Create necessary directories
echo "Creating website directory structure..."
sudo mkdir -p "$WEB_ROOT"
sudo mkdir -p "$LOG_DIR"
sudo chown -R root:root "$WEB_ROOT"
sudo chmod -R 755 "$WEB_ROOT"

# Setup Virtual Host file
echo "Creating Apache Virtual Host configuration..."
sudo tee "$VHOST_CONF" > /dev/null <<EOL
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName $MYDOMAIN
    ServerAlias www.$MYDOMAIN
    DocumentRoot $WEB_ROOT
    ErrorLog \${APACHE_LOG_DIR}/error.log
    CustomLog \${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOL

# Enable the site and restart Apache
echo "Enabling the virtual host and restarting Apache..."
sudo a2ensite "${MYDOMAIN}.conf"
sudo systemctl restart apache2

# Wait for the domain to resolve
echo "Checking if domain resolves..."
while ! nc -z "$MYDOMAIN" 80; do
    echo "Waiting 2.5 minutes for DNS resolution..."
    sleep 150
done
echo "Domain appears to resolve. Proceeding with SSL setup..."

# Obtain an SSL certificate via Certbot
CERTBOT_CMD="sudo certbot --apache --email admin@$MYDOMAIN --agree-tos --no-eff-email -d $MYDOMAIN --redirect --hsts --uir"
echo "Running Certbot command: $CERTBOT_CMD"
$CERTBOT_CMD || {
    echo "Certbot failed! You may need to manually retry with:"
    echo "  $CERTBOT_CMD"
    exit 1
}

echo "SSL certificate setup complete."

# Restart Apache to apply changes
echo "Final Apache restart..."
sudo systemctl restart apache2

echo "Domain setup completed successfully for: $MYDOMAIN"
exit 0
```

Run it, passing your domain as an argument:

```bash
sudo bash domain-setup.sh yourdomain.com
```

This script creates the web root and log directories, writes out a virtual host config, waits until the domain is actually resolving on port 80, then hands things off to Certbot to request and install a real Let's Encrypt certificate  complete with HSTS and automatic HTTP-to-HTTPS redirection.

Once Certbot finishes, hit your domain over HTTPS and check the certificate in your browser it should show as valid and trusted, issued by Let's Encrypt.

<figure><img src="../.gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

At this point we have a clean, hardened Apache install, sitting on a domain we control, serving over a properly trusted TLS certificate, and returning a default 404 to anything that hits it without the right request. That's our base layer.

<figure><img src="../.gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>

### Step 4: C2 Server Setup

As we have installed every thing now let's go to our Apache config again and set few things up <br>

* First Let's Setup the proxy that when ever a request come it goes to our c2 server for that we'll use ProxyPass and ProxyPassReverse (file location of config): `/etc/apache2/sites-available/domainname-le-ssl.conf`

```
SSLProxyEngine On
SSLProxyVerify None
SSLProxyCheckPeerCN Off
SSLProxyCheckPeerName Off
SSLProxyCheckPeerExpire Off
ProxyPreserveHost On
RewriteEngine On

ProxyPass / https://<C2 server ip>:4443/
ProxyPassReverse / https://<C2 server ip>:4443/
```

> What is ProxyzPass and ProxyPassReverse:- The ProxyPass and ProxyPassReverse allow for all traffic to be sent to the Cobalt Strike server over HTTPS.

To save our changes and apply the new configuration file to Apache, we must restart the server. This can be done using the service command:

```
service apache2 restart 
```

Now Let's head to our C2 and test the redirector setup, we can now test our new domain front by creating an HTTPS listener from the Cobalt Strike client and pointing the host to our domain.

<figure><img src="../.gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

and now let's create a https listener&#x20;

<figure><img src="../.gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

and now as of we have create the listener now let's go and visit our web site and see if we get any web logs in our C2&#x20;

<figure><img src="../.gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

as we can see our server is working fine now let's try to get our own beacon&#x20;

<figure><img src="../.gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

and now the movement of truth&#x20;

<figure><img src="../.gitbook/assets/image (57).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/867733100925550592.webp" alt=""><figcaption></figcaption></figure>

Next Article will be on how to hard in the server and use custom headers
