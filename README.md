# JellyfinApache2Proxy
## A Solution to Proxy Jellyfin through an Apache2 Virtual Host as a Directory on an Existing Apache2 server.
rev. Jack Dobson, The Church of neoReality

## Intention
To Proxy a Jellyfin server as a directory on an existing Apache2 server so that is the only method of accessing the Jellyfin 
server from the network.

## Prerequisites
Understand that I started this endevor using the default Flatpak installation of Jellyfin available from the Mint Linux Software
Manager App. That was, as it always seems to be, a mistake. My opinion of Flatpak is pretty well cemented in the, if it's only
available as a Flatpak consider a different solution, camp.

Step 1 of this process is to unisntall that noise, & install from the Jellyfin apt repo https://jellyfin.org/downloads/server as 
the instructions work on it, but from what I saw early on, not on the Flatpak. I have not gone back to test if that is still the 
case with my finished solution... and I don't intend to. Go nuts!

## Name Resolution is Essential 'It is ALWAYS a DNS issue.'
This guide assumes you have running Jellyfin & Apache2 servers, & want to proxy the direct connection from the Jellyfin port 8096
to a directory /jellyfin on the Apache2 server. 

ATTENTION!, Here and Now... before you come screaming at me... direct connections to port 8096 may no longer work in my configuration, as it is my intention to make damn well sure they don't as a matter of security policy. My server responds to one thing, HTTPS.

In this example the server (in this case a VMWare Workstation Pro Virtual Machine) named myserver @ 10.1.10.250, I'm going to 
create a jellyfin.local Apache2 Virtual Host to Proxy to.

Both Virtual hosts must be resolvable by the server.

** /etc/hosts example **
```
## loopback, obviously
127.0.0.1       localhost

## network resolvable webserver address. Of course all clients will have to be able to resolve the webserver.
10.1.10.250     myserver.local  myserver

## Jellyfin Virtual Host address. ONLY needs to be resolvable from the Apache server.
## Clients don't need to be able to resolve this address.
10.1.10.250     jellyfin.local  jellyfin

## IPV6 nonsense, as the entire industry jumped the gun.
## IPV6 is disabled, from grub up.
## The following lines are desirable for IPv6 capable hosts
## ::1     ip6-localhost ip6-loopback
## fe00::0 ip6-localnet
## ff00::0 ip6-mcastprefix
## ff02::1 ip6-allnodes
## ff02::2 ip6-allrouters
```
Clients will have to be able to resolve the Apache2 server's address (myserver), ONLY the server needs to be able
to resolve the jellyfin.local virtual host. In this example when you surf to https://myserver.local/jellyfin from any client, the
Apache2 server will proxy the Jellyfin server, that the clients have no knowledge of or direct access to, as a directory on
your existing server.

## Jellyfin Server Configuration
This guide assumes you can currently connect directly to your existing Jellyfin server on port 8096.

On the Dashboard Networking settings. 
```
Set Base URl to: /jellyfin
Bind to local network address: 10.1.10.250
Set your LAN scope: 10.1.10.0/24
Set Known Proxies to the server IP address: 10.1.10.250
```
*Replacing the relevant info from your network, of course.

Save the settings.

## Apache2 Configuration 
These Apache configuration files maintain the default naming and configuration of the vanillia Apache files 
as much as possible. Explaining standard Apache2 sites-enabled is beyond the scope of this document.

** /etc/apache2/sites-available/000-default.conf **
```
## Main server virtual host. 
<VirtualHost myserver.local:80>

ServerName myserver.local

## Notice the document root is set to the default Apache2 directory.
## The index.html in this directory is the default Apache page, or the directory for your existing website.
DocumentRoot /var/www/html

## HTTP is bad, M'kay? Unencrypted traffic simply should not be.
## HTTP to HTTPS redirect
Redirect permanent / https://myserver.local/

ErrorLog ${APACHE_LOG_DIR}/000-default-error.log
CustomLog ${APACHE_LOG_DIR}/000-default-access.log combined

</VirtualHost>
```
** /etc/apache2/sites-available/default-ssl.conf **
```
## Main server SSL virtual host. 
<VirtualHost myserver.local:443>

ServerName myserver.local

## Notice the document root is set to the default Apache2 directory.
## The index.html in this directory is the default Apache page, or the directory for your existing website.
DocumentRoot /var/www/html

ErrorLog ${APACHE_LOG_DIR}/default-ssl-error.log
CustomLog ${APACHE_LOG_DIR}/default-ssl-access.log combined

SSLEngine on
Protocols h2 http/1.1
ProxyPreserveHost On

## Tell Server to forward requests that came from TLS connections
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Port "443"

## This is where the MAGIC happens on the main server to proxy
## calls to the myserver.local/jellyfin directory to the jellyfin server.

ProxyPass "/jellyfin/" "ws://jellyfin.local:8096/"
ProxyPassReverse "/jellyfin/" "ws://jellyfin.local:8096/"

ProxyPass "/jellyfin/" "https://jellyfin.local:8096/"
ProxyPassReverse "/jellyfin/" "https://jellyfin.local:8096/"

ProxyPass "/jellyfin" "ws://jellyfin.local:8096/"
ProxyPassReverse "/jellyfin" "ws://jellyfin.local:8096/"

ProxyPass "/jellyfin" "https://jellyfin.local:8096/"
ProxyPassReverse "/jellyfin" "https://jellyfin.local:8096/"

## This is a simple self-signed example, your ssl setup here, basically.
SSLCertificateFile /etc/ssl/certs/myserver.local.crt
SSLCertificateKeyFile /etc/ssl/private/myserver.local.key

<FilesMatch "\.(?:cgi|shtml|phtml|php)$">
SSLOptions +StdEnvVars
</FilesMatch>

<Directory /usr/lib/cgi-bin>
SSLOptions +StdEnvVars
</Directory>

</VirtualHost>
```
** /etc/apache2/sites-available/jellyfin.conf **
```
## Probably superfluous HTTP redirect... but I like it for completeness.
<VirtualHost jellyfin.local:80>

ServerName jellyfin.local

## ATTENTION!, Here and Now...
## Notice the document root is set to a jellyfin only subdir, outside of the main server's scope.
## /var/www/jellyfin is simply an empty directory that is owned by jellyfin:jellyfin.
DocumentRoot /var/www/jellyfin

## Unencryped traffic should not exist.
## HTTP to HTTPS redirect
Redirect permanent / https://jellyfin.local/

ErrorLog /var/log/apache2/jellyfin.local-error.log
CustomLog /var/log/apache2/jellyfin.local-access.log combined

</VirtualHost>

## Jellyfin server SSL virtual host. 
<IfModule mod_ssl.c>
<VirtualHost jellyfin.local:443>

ServerName jellyfin.local

## ATTENTION!, Here and Now...
## Notice the document root is set to a jellyfin only subdir, outside of the main server's scope.
## /var/www/jellyfin is simply an empty directory that is owned by jellyfin:jellyfin.
DocumentRoot /var/www/jellyfin

ProxyPreserveHost On

## Tell Jellyfin to forward requests that came from TLS connections
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Port "443"

## Proxy the Jellyfin server port 8096 to the root / of jellyfin.local
ProxyPass "/" "ws://jellyfin.local:8096/"
ProxyPassReverse "/" "ws://jellyfin.local:8096/"

ProxyPass "/" "https://jellyfin.local:8096/"
ProxyPassReverse "/" "https://jellyfin.local:8096/"

SSLEngine on

## ATTENTION!, Here and Now...
## jellyfin.local has it's own certs as expected by the Jellyfin server.
SSLCertificateFile /etc/ssl/certs/jellyfin.local.crt
SSLCertificateKeyFile /etc/ssl/private/jellyfin.local.key

Protocols h2 http/1.1

## Enable only strong encryption ciphers and prefer versions with Forward Secrecy
SSLCipherSuite HIGH:RC4-SHA:AES128-SHA:!aNULL:!MD5
SSLHonorCipherOrder on

## Disable insecure SSL and TLS versions
SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

ErrorLog /var/log/apache2/jellyfin.local-ssl-error.log
CustomLog /var/log/apache2/jellyfin.local-ssl-access.log combined

</VirtualHost>
</IfModule>
```
Remember to create the /var/www/jellyfin directory and give it to Jellyfin.
```
sudo mkdir -p /var/www/jellyfin && sudo chown jellyfin:jellyfin /var/www/jellyfin;
```
With this configuration you should be able to surf to https://jellyfin.local/ from your Webserver to directly
configure the Jellyfin server. As well as directly to the IP address:port (I think).

The Proxied Jellyfin server should be availble from https://myserver.local/jellyfin as if it is a directory on your existing website.

*Adjusting for your network values, of course.

Once again, before you come screaming at me... direct connections to 8096 may no longer work as it is my intention to make
direct connections to the Jellyfin server intentionally unavailable as a security policy.

## Conclusion

This solution should provide a secure and non-invasive method for proxying a Jellyfin server proxy with little to no impact 
on an existing Apache2 configuration. As far as I can tell so far.

Please feel free to implement this solution at your convenience should it suit your needs. Comments are welcome.

rev. Jack Dobson
