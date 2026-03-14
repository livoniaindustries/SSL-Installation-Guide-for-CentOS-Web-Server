# SSL Installation Guide for CentOS Web Server

**🔒 About SSL:** This guide covers installing and configuring SSL/TLS encryption on a CentOS web server using mod\_ssl and OpenSSL. SSL certificates encrypt traffic between your server and visitors, securing sensitive data and improving trust.

## Prerequisites

**Note:** Before starting, ensure you have a CentOS server with Apache (httpd) installed or you'll install it during this process.

## Step-by-Step SSL Installation

### Step 1: Install Apache (if not already installed)

\# Install Apache sudo yum install httpd -y # Start Apache and enable auto-start sudo systemctl start httpd sudo systemctl enable httpd.service # Check Apache status sudo systemctl status httpd

### Step 2: Configure Firewall for HTTPS

\# Allow HTTPS through firewall sudo firewall-cmd --permanent --add-service=https # Also ensure HTTP is allowed (for redirects) sudo firewall-cmd --permanent --add-service=http # Reload firewall to apply changes sudo firewall-cmd --reload # Verify firewall rules sudo firewall-cmd --list-all

### Step 3: Install mod\_ssl and OpenSSL

\# Install mod\_ssl and OpenSSL packages sudo yum install mod\_ssl openssl -y

### Step 4: Create SSL Certificate Directories

\# Create directory for private keys sudo mkdir -p /etc/ssl/private # Set proper permissions sudo chmod 700 /etc/ssl/private

### Step 5: Generate Self-Signed SSL Certificate

\# Generate SSL key files (valid for 365 days) sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \\ -keyout /etc/ssl/private/apache-selfsigned.key \\ -out /etc/ssl/certs/apache-selfsigned.crt

You will be prompted to enter information about your site. The most important field is **Common Name** - enter your domain name (e.g., example.com or www.example.com).

🔐 OpenSSL Certificate Generation - Fill in your site information  
Common Name (e.g., server FQDN or YOUR domain name) \[\]: example.com

#### OpenSSL prompts explained:

| Field | Description | Example |
| --- | --- | --- |
| Country Name | Two-letter country code | US, GB, RU, etc. |
| State or Province | Full state/province name | California |
| Locality Name | City name | San Francisco |
| Organization Name | Company/organization name | Example Inc |
| Organizational Unit | Department name | IT Department |
| Common Name | **Domain name (most important!)** | example.com |
| Email Address | Admin email | admin@example.com |

### Step 6: Configure Perfect Forward Secrecy (PFS)

\# Generate Diffie-Hellman parameters (takes a few minutes) sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048 # Append DH parameters to certificate cat /etc/ssl/certs/dhparam.pem | sudo tee -a /etc/ssl/certs/apache-selfsigned.crt

**What is PFS?** Perfect Forward Secrecy ensures that session keys are not compromised even if the server's private key is compromised. This is an important security feature for modern websites.

### Step 7: Configure Apache SSL Settings

Edit the SSL configuration file:

sudo vi /etc/httpd/conf.d/ssl.conf

Find and update the following lines:

\# Change ServerName to your domain (replace example.com with your actual domain) ServerName www.example.com:443 # Comment out or remove the default certificate lines: # SSLCertificateFile /etc/pki/tls/certs/localhost.crt # SSLCertificateKeyFile /etc/pki/tls/private/localhost.key # Add your new certificate paths: SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

#### Additional SSL hardening (optional but recommended):

\# Add these lines to improve SSL security SSLCipherSuite HIGH:!aNULL:!MD5 SSLProtocol All -SSLv3 -TLSv1 -TLSv1.1 SSLCompression off SSLSessionTickets off # Add DH parameters (if not already appended to certificate) SSLOpenSSLConfCmd DHParameters "/etc/ssl/certs/dhparam.pem"

### Step 8: Configure VirtualHost for HTTPS

In the same `ssl.conf` file or in your virtual host configuration, ensure you have a VirtualHost block for port 443:

<VirtualHost \*:443> ServerName example.com ServerAlias www.example.com DocumentRoot /var/www/html SSLEngine on SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key <Directory /var/www/html> Options FollowSymLinks AllowOverride All Require all granted </Directory> ErrorLog logs/ssl\_error\_log TransferLog logs/ssl\_access\_log LogLevel warn </VirtualHost>

### Step 9: Restart Apache and Test SSL

\# Test Apache configuration for syntax errors sudo httpd -t # Restart Apache to apply changes sudo systemctl restart httpd.service # Check Apache status sudo systemctl status httpd

After restarting, test your SSL certificate by visiting your domain with HTTPS:

https://your-domain.com

**⚠️ Browser Warning:** Since this is a self-signed certificate, browsers will show a security warning. This is normal for self-signed certificates. For production sites, consider using Let's Encrypt for free trusted certificates.

## Redirect HTTP to HTTPS

To automatically redirect all HTTP traffic to HTTPS, add this configuration to your `.htaccess` file or Apache configuration:

### Method 1: Using .htaccess

RewriteEngine On RewriteCond %{HTTPS} !on RewriteRule (.\*) https://%{HTTP\_HOST}%{REQUEST\_URI} \[R=301,L\]

### Method 2: Using Apache VirtualHost (HTTP)

<VirtualHost \*:80> ServerName example.com ServerAlias www.example.com # Redirect all traffic to HTTPS RewriteEngine On RewriteCond %{HTTPS} off RewriteRule (.\*) https://%{HTTP\_HOST}%{REQUEST\_URI} \[R=301,L\] </VirtualHost>

## Troubleshooting: HTTP to HTTPS Redirect Not Working

❓ Problem: HTTP to HTTPS redirect is not working - site opens via HTTP even though SSL is installed

**Solution:** Add this redirect rule to your .htaccess file:

RewriteEngine On RewriteCond %{HTTPS} !on RewriteRule (.\*) https://%{HTTP\_HOST}%{REQUEST\_URI} \[R=301,L\]

Make sure:

*   mod\_rewrite is enabled
*   .htaccess files are allowed (AllowOverride All in Apache config)
*   The rule is placed before any other rules
*   Clear browser cache after adding the rule

## Complete SSL Configuration Example

Here's a complete working example of `/etc/httpd/conf.d/ssl.conf`:

Listen 443 https <VirtualHost \*:443> ServerName example.com ServerAlias www.example.com DocumentRoot /var/www/html SSLEngine on SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key # Modern SSL configuration SSLProtocol All -SSLv3 -TLSv1 -TLSv1.1 SSLCipherSuite HIGH:!aNULL:!MD5 SSLCompression off SSLSessionTickets off <Directory /var/www/html> Options FollowSymLinks AllowOverride All Require all granted </Directory> ErrorLog logs/example.com-ssl-error\_log CustomLog logs/example.com-ssl-access\_log common </VirtualHost>

## Using Let's Encrypt (Free Trusted Certificates)

**📌 Alternative:** For production sites, it's better to use Let's Encrypt for free, automatically renewed, browser-trusted certificates.

\# Install Certbot (Let's Encrypt client) sudo yum install epel-release -y sudo yum install certbot python2-certbot-apache -y # Obtain and install SSL certificate sudo certbot --apache -d example.com -d www.example.com # Test auto-renewal sudo certbot renew --dry-run

## SSL Verification Commands

| Command | Description |
| --- | --- |
| `sudo openssl x509 -in /etc/ssl/certs/apache-selfsigned.crt -text -noout` | View certificate details |
| `echo \| openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \| openssl x509 -text` | Check SSL certificate from remote server |
| `sudo httpd -t` | Test Apache configuration |
| `sudo systemctl status httpd` | Check Apache status |
| `sudo tail -f /var/log/httpd/ssl_error_log` | Monitor SSL error logs |

## SSL Security Checklist

| Item | Status |
| --- | --- |
| HTTPS redirect working (HTTP → HTTPS) | ✅ / ❌ |
| No mixed content (all resources loaded over HTTPS) | ✅ / ❌ |
| SSL certificate valid and not expired | ✅ / ❌ |
| Strong cipher suites configured | ✅ / ❌ |
| HSTS enabled (HTTP Strict Transport Security) | ✅ / ❌ |
| Firewall allows HTTPS (port 443) | ✅ / ❌ |

## Enable HSTS (Optional but Recommended)

Add this to your HTTPS VirtualHost configuration to tell browsers to always use HTTPS:

<VirtualHost \*:443> # ... existing configuration ... # Enable HSTS Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains" </VirtualHost>

**⚠️ Caution:** Enable HSTS only after confirming HTTPS works perfectly everywhere on your site, as it forces browsers to use HTTPS for the specified time period.

## Common Issues and Solutions

| Issue | Solution |
| --- | --- |
| Apache won't start after SSL configuration | Run `sudo httpd -t` to check for syntax errors. Check file paths in ssl.conf. |
| Certificate works but browser shows warning | This is normal for self-signed certificates. Use Let's Encrypt for trusted certs. |
| HTTPS works but redirect loops | Check your redirect rules for conflicts. Ensure you're not redirecting both ways. |
| Mixed content warnings | Update all resource URLs (images, CSS, JS) to use HTTPS or relative paths. |
| SSL certificate expired | For self-signed: generate new one. For Let's Encrypt: run `sudo certbot renew`. |

## Quick Reference Commands

| Action | Command |
| --- | --- |
| Generate self-signed cert | `sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt` |
| Generate DH parameters | `sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048` |
| Restart Apache | `sudo systemctl restart httpd` |
| Check SSL certificate | `sudo openssl x509 -in /etc/ssl/certs/apache-selfsigned.crt -text -noout` |
| Test HTTPS redirect | `curl -I http://example.com` |
