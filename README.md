# Notes on certbot

My home network got shrunk to one IP address.  My WOW cable service used to give me to IPs, and they've changed to only one IP address.  Thus, all my web sites need to share a common IP address.  Since I was using cert-manager on my kubernetes cluster and some of my websites on separate hosts, I need to create a separate reverse proxy server that takes port 80 and 443 traffic.

For the port 80 traffic, I want to redirect it to port 443.  And, for my 443 traffic, I want to terminate the SSL, and redirect the unencrypted traffic to the appropriate web server half of which are on my k8s cluster.  Thus I need to retire cert-manager on mhy k8s cluster and setup certbot on my reverse proxy.  

Here's an example of an apache virtual server that currently works on port 80 and redirects to my k8s cluster (http://192.168.100.176:30140):

```
[root@dell1 sites-enabled]# cat proxy.sancapweather.com.conf
<VirtualHost *:80>
 ServerName sancapweather.com
 ProxyPreserveHost On
 ProxyPass / http://192.168.100.174:30140/
 ProxyPassReverse / http://192.168.100.174:30140/
   ErrorLog logs/sancapweather.com-error_log
   CustomLog logs/sancapweather.com-access_log combined
</VirtualHost>
[root@dell1 sites-enabled]#
```
I want to enable https://sancapweather.com, thus I install and run certbot and get a let's encrypt certificate.  I am already using certbot; my problem is to get all SSL processing done in one place... I am down to only one IP address.

One the host that is running my apache reverse proxy:
```
[root@dell1 sites-enabled]# certbot --apache
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator apache, Installer apache
Starting new HTTPS connection (1): acme-v02.api.letsencrypt.org

Which names would you like to activate HTTPS for?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: camptonhillsweather.net
2: camptonhillsweather.com
3: jackkozik.com
4: www.jackkozik.com
5: liz.kozik.net
6: www.liz.kozik.net
7: kozikfamily.net
8: kozikfamily.com
9: www.kozikfamily.com
10: www.kozikfamily.net
11: napervilleweather.com
12: napervilleweather.net
13: sancapweather.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel): 13
Requesting a certificate for sancapweather.com
Performing the following challenges:
http-01 challenge for sancapweather.com
Waiting for verification...
Cleaning up challenges
Created an SSL vhost at /etc/httpd/sites-enabled/proxy.sancapweather.com-le-ssl.conf
Deploying Certificate to VirtualHost /etc/httpd/sites-enabled/proxy.sancapweather.com-le-ssl.conf
Redirecting vhost in /etc/httpd/sites-enabled/proxy.sancapweather.com.conf to ssl vhost in /etc/httpd/sites-enabled/proxy.sancapweather.com-le-ssl.conf

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations! You have successfully enabled https://sancapweather.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/sancapweather.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/sancapweather.com/privkey.pem
   Your certificate will expire on 2022-03-08. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again with the "certonly" option. To non-interactively
   renew *all* of your certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
[root@dell1 sites-enabled]#
```
I was pleased to see that certbot very nicely edited my .conf above and created a new one to handle the 443 traffic. 
```
[root@dell1 sites-enabled]# cat proxy.sancapweather.com.conf
<VirtualHost *:80>
 ServerName sancapweather.com
 ProxyPreserveHost On
 ProxyPass / http://192.168.100.174:30140/
 ProxyPassReverse / http://192.168.100.174:30140/
   ErrorLog logs/sancapweather.com-error_log
   CustomLog logs/sancapweather.com-access_log combined
RewriteEngine on
RewriteCond %{SERVER_NAME} =sancapweather.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>

[root@dell1 sites-enabled]# cat proxy.sancapweather.com-le-ssl.conf
<IfModule mod_ssl.c>
<VirtualHost *:443>
 ServerName sancapweather.com
 ProxyPreserveHost On
 ProxyPass / http://192.168.100.174:30140/
 ProxyPassReverse / http://192.168.100.174:30140/
   ErrorLog logs/sancapweather.com-error_log
   CustomLog logs/sancapweather.com-access_log combined
SSLCertificateFile /etc/letsencrypt/live/sancapweather.com/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/sancapweather.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
SSLCertificateChainFile /etc/letsencrypt/live/sancapweather.com/chain.pem
</VirtualHost>
</IfModule>
[root@dell1 sites-enabled]#
```
And now, I verify that https://sancapweather.com works and that http://sancapweather.com redirects to https://sancapweather.com

This was easier than I thought.  

# Move to Cloudflare
With our move to a new a ISP, we had to re-do our certbot approach. Our ISP blocks port 80 and the HTTP-01 challenge for certbot will not work.
Instead we must use the --preferred-challenge dns option.  With this option, certbot prompts with two different TXT strings to put into the DNS entry for the domain being managed.  The certificate is renewed when the tool verifies that the DNS TXT records are found, thus proving ownership.

This prompting of a TXT string and editting it into the DNS record can be done manually, but with Cloudflare DNS one can install a plugin to certbot that will do this automatically.  As part of doing this, I setup a cloudflare.ini file with a Global API key from my Cloudflare login. 
## Cloudflare / cerbot renew references
- https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins
- https://certbot-dns-cloudflare.readthedocs.io/en/stable/
- https://installati.one/centos/7/python2-certbot-dns-cloudflare/

## Renewing kozik.net 
```
[root@dell1 certbot]# certbot certonly --dns-cloudflare --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini -d "*.kozik.net" -d "kozik.net"
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-cloudflare, Installer None
Starting new HTTPS connection (1): acme-v02.api.letsencrypt.org
Cert is due for renewal, auto-renewing...
Renewing an existing certificate for *.kozik.net and kozik.net
Performing the following challenges:
dns-01 challenge for kozik.net
dns-01 challenge for kozik.net
Starting new HTTPS connection (1): api.cloudflare.com
Starting new HTTPS connection (1): api.cloudflare.com
Waiting 10 seconds for DNS changes to propagate
Waiting for verification...
Cleaning up challenges
Starting new HTTPS connection (1): api.cloudflare.com
Starting new HTTPS connection (1): api.cloudflare.com

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/kozik.net/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/kozik.net/privkey.pem
   Your certificate will expire on 2022-11-12. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again. To non-interactively renew *all* of your
   certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
## Sub-sub Domains
I want to wild card SSL access to domains https://*.k8s.kozik.net. I have a kubernetes cluster, and I want to grant external access to certain test services.  These test services will everntually get their own dedicated sub domain, eg https://mynewservice.kozik.net, but for quick testing I need to create a sub-sub domain.  

It turns out that the certificate for kozik.net was created with "Domains:  kozik.net, *.kozik.net".  A certificate like that will generate a browswer error and fail the [https://https://www.sslshopper.com/ssl-checker.html] for anything in the sub-subdomain *.k8s.kozik.net.  For example,

![image](https://github.com/user-attachments/assets/89655d29-366b-4102-8d84-1fb23e10000f)

Checking the references on the [Let's Encrypt Community](https://community.letsencrypt.org/), I found a thread [Certificates for sub.subs.domian](https://community.letsencrypt.org/t/certificates-for-sub-subs-domian/107493) that explained that one needs to expand the certificate to enumerate each of the sub-sub domains that need to get wildcarded.  In my case, I needed to add *.k8s.kozik.net to my kozik.net certificate.

Here's the command lines that I ran to do that:
```
[root@dell1 certbot]# certbot certonly --dns-cloudflare --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini --dns-cloudflare-propagation-seconds 30 -d "*.kozik.net" -d "kozik.net" -d "*.k8s.kozik.net"

Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-cloudflare, Installer None
Starting new HTTPS connection (1): acme-v02.api.letsencrypt.org

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
You have an existing certificate that contains a portion of the domains you
requested (ref: /etc/letsencrypt/renewal/kozik.net.conf)

It contains these names: *.kozik.net, kozik.net

You requested these names for the new certificate: *.kozik.net, kozik.net,
*.k8s.kozik.net.

Do you want to expand and replace this existing certificate with the new
certificate?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(E)xpand/(C)ancel: E
Renewing an existing certificate for *.kozik.net and 2 more domains
Performing the following challenges:
dns-01 challenge for kozik.net
dns-01 challenge for kozik.net
Starting new HTTPS connection (1): api.cloudflare.com
Starting new HTTPS connection (1): api.cloudflare.com
Waiting 30 seconds for DNS changes to propagate
Waiting for verification...
Cleaning up challenges
Starting new HTTPS connection (1): api.cloudflare.com
Starting new HTTPS connection (1): api.cloudflare.com

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/kozik.net/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/kozik.net/privkey.pem
   Your certificate will expire on 2024-12-08. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again. To non-interactively renew *all* of your
   certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

[root@dell1 certbot]#  certbot certificates
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Found the following certs:
  
  Certificate Name: kozik.net
    Serial Number: 34c30d2a91b3e2b08d628e8670ae2d011f4
    Key Type: RSA
    Domains: *.kozik.net *.k8s.kozik.net kozik.net
    Expiry Date: 2024-12-08 15:52:52+00:00 (VALID: 89 days)
    Certificate Path: /etc/letsencrypt/live/kozik.net/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/kozik.net/privkey.pem

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
[root@dell1 certbot]#
```
A couple of notes:  
- I am using the cloudflare DNS server and it has [a special module](https://certbot-dns-cloudflare.readthedocs.io/en/stable/) that automates the auto-renewal
- The command didn't work the first time for me. I had to re-rerun it, inserting a 30 second pause.  

# References
- https://certbot.eff.org/instructions?ws=apache&os=centosrhel7
- https://linuxhostsupport.com/blog/how-to-install-lets-encrypt-on-centos-7-with-apache/
- https://eff-certbot.readthedocs.io/en/stable/using.html#certbot-commands
- [Welcome to certbot-dns-cloudflare’s documentation!](https://certbot-dns-cloudflare.readthedocs.io/en/stable/)

