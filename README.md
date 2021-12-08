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

# References
- https://certbot.eff.org/instructions?ws=apache&os=centosrhel7
- https://linuxhostsupport.com/blog/how-to-install-lets-encrypt-on-centos-7-with-apache/
- https://eff-certbot.readthedocs.io/en/stable/using.html#certbot-commands
- 

