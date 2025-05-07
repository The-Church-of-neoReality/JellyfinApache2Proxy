# JellyfinApache2Proxy
## A Solution to Proxy Jellyfin through an Apache2 Virtual Host as a Directory on an Existing Apache2 server.
rev. Jack Dobson, The Church of neoReality

## Intention
To Proxy a Jellyfin server as a directory on an existing Apache2 VirtualHost; https://<SERVER_NAME>/jellyfin, With minimal
alteration of the existing webserver config/network. All things should be as simple as possible and no more simple... or
Keep it simple Stupid. What's a vamillia Apache2 server come with? A config for port 80 and one for port 443. So, it looks
like the Apache2 config is port based... lean into it and go with the flow. A config for port 8096. There's some Sun Tzu in 
there somewhere.

## Prerequisites
Understand that I started this endevor using the default Flatpak installation of Jellyfin available from the Mint Linux Software
Manager App. That was, as it always seems to be, a mistake. My opinion of Flatpak is pretty well cemented in the, if it's only
available as a Flatpak consider a different solution, camp.

Step 1 of this process is to install from the Jellyfin apt repo https://jellyfin.org/downloads/server as these instructions are for 
it, as were all the instructions I combed through on the web to come to this solution, but from what I saw early on, they did not work 
on the Flatpak. I have not gone back to test if that is still the case with my finished solution... and I don't intend to. Go nuts!

## Name Resolution is Essential 'It is ALWAYS a DNS issue.'
This guide assumes you have running Jellyfin & Apache2 servers, & want to proxy the direct connection from the Jellyfin port 8096
to a directory /jellyfin on the Apache2 server. 

ATTENTION!, Here and Now... before you come screaming at me... direct connections to port 8096 may no longer work in my 
configuration, as it is my intention to make damn well sure they don't as a matter of security policy. My server responds to one 
thing, HTTPS.

Both Virtual hosts must be resolvable by the Apache server.

** /etc/hosts example **
```
## loopback, obviously
127.0.0.1       localhost

## network resolvable webserver address. Of course all clients will have to be able to resolve the webserver.
10.1.10.250     myserver.local  myserver

## Jellyfin Virtual Host address. ONLY needs to be resolvable from the Apache server.
## Clients don't need to be able to resolve this address. In my case it's the same IP.
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
This guide assumes you CAN currently connect directly to your existing Jellyfin server on port 8096. Love you Alanis.

On the Dashboard:
General: 
```
Set your server name to what you put in the server's hosts file to identify the Jellyfin server Virtual Host. 
```
Networking:
```
Set Base URl to: /jellyfin
Bind to local network address: 10.1.10.250
Set your LAN scope: 10.1.10.0/24
Set Known Proxies to the server IP address: 10.1.10.250
```
*Replacing the relevant info from your network, of course.

Save the settings. **Remembering that, damn, the save button is all the way down at the end of the page.

## Apache2 Configuration
These Apache configuration files maintain the default naming and configuration of the vanillia Apache files 
as much as possible, while including annotation & explanation.

** /etc/apache2/sites-available/000-default.conf **
```
## 000-default.conf
## Main server virtual host. Responds to everything on port 80 
<VirtualHost *:80>

ServerName myserver.local
ServerAdmin me@wherever.org

## Notice the document root is set to the default Apache2 directory.
## The index.html in this directory is the default Apache page.
## or this is where your existing website is being run from.
DocumentRoot /var/www/html

## HTTP is bad, M'kay? Unencrypted traffic simply should not be.
## HTTP to HTTPS redirect
Redirect permanent / https://myserver.local/

</VirtualHost>

```
** /etc/apache2/sites-available/default-ssl.conf **
```
## default-ssl.conf
## Main server SSL virtual host. Responds to everything on port 443.
<VirtualHost *:443>

ServerName myserver.local
ServerAdmin me@wherever.org

## Notice the document root is set to the default Apache2 directory.
## The index.html in this directory is the default Apache page.
DocumentRoot /var/www/html

SSLEngine on
Protocols h2 http/1.1

## Tell Server to forward requests that came from TLS connections
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Port "443"

## This is a simple self-signed example, your ssl setup here, basically.
SSLCertificateFile /etc/ssl/certs/myserver.local.crt
SSLCertificateKeyFile /etc/ssl/private/myserver.local.key

## Proxy the directory on main server to the virtual host for the jellyfin server.
ProxyAddHeaders On
ProxyPreserveHost On 

## jellyfin.local in this example is whatever you used to idetify your jellyfin server in the hosts file.
ProxyPass "/jellyfin" "ws://jellyfin.local:8096/jellyfin"
ProxyPassReverse "/jellyfin" "ws://jellyfin.local:8096/jellyfin"
ProxyPass "/jellyfin" "https://jellyfin.local:8096/jellyfin"
ProxyPassReverse "/jellyfin" "https://jellyfin.local:8096/jellyfin"

ProxyPass "/jellyfin/" "ws://jellyfin.local:8096/jellyfin"
ProxyPassReverse "/jellyfin/" "ws://jellyfin.local:8096/jellyfin"
ProxyPass "/jellyfin/" "https://jellyfin.local:8096/jellyfin"
ProxyPassReverse "/jellyfin/" "https://jellyfin.local:8096/jellyfin"

## Now *:443/jellyfin will be proxied to the Apache Virtual Host for the Jellyfin server.

## What follows is the end of the vanillia default-ssl.conf
<FilesMatch "\.(?:cgi|shtml|phtml|php)$">
SSLOptions +StdEnvVars
</FilesMatch>

<Directory /usr/lib/cgi-bin>
SSLOptions +StdEnvVars
</Directory>

</VirtualHost>

```
** /etc/apache2/sites-available/420-port8096.conf **
```
## 420-port8096.conf
## Jellyfin server SSL virtual host. That responds to everything on port 8096.
<IfModule mod_ssl.c>
<VirtualHost *:8096>

## or whatever you used in the hosts file.
ServerName jellyfin.local
ServerAdmin me@wherever.org

## !!!Notice!!! the document root is set to a Jellyfin only subdir, outside of the main server's scope.
## /var/www/jellyfin is empty and owned by jellyfin:jellyfin, so Apache has a legitimate directory to point to.
DocumentRoot /var/www/jellyfin

SSLEngine on
Protocols h2 http/1.1

## Tell Jellyfin to forward requests that came from TLS connections
RequestHeader set X-Forwarded-Proto "https"
RequestHeader set X-Forwarded-Port "443"

## This is a simple self-signed example. Should be replaced with your SSL solution.
## Jellyfin server expects certs to match its hostname...?
SSLCertificateFile /etc/ssl/certs/jellyfin.local.crt
SSLCertificateKeyFile /etc/ssl/private/jellyfin.local.key

## Proxy the ROOT / of the Jellyfin Virtual Host to port 8096.
ProxyAddHeaders On
ProxyPreserveHost On

ProxyPass "/" "ws://jellyfin.local:8096"
ProxyPassReverse "/" "ws://jellyfin.local:8096"
ProxyPass "/" "https://jellyfin.local:8096"
ProxyPassReverse "/" "https://jellyfin.local:8096"

## Enable only strong encryption ciphers and prefer versions with Forward Secrecy
SSLCipherSuite HIGH:RC4-SHA:AES128-SHA:!aNULL:!MD5
SSLHonorCipherOrder on

## Disable insecure SSL and TLS versions
SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

</VirtualHost>
</IfModule>

```
Remember to create the /var/www/jellyfin directory and give it to Jellyfin.
```
sudo mkdir -p /var/www/jellyfin && sudo chown jellyfin:jellyfin /var/www/jellyfin;
```
The Proxied Jellyfin Virtual Host should respond to https://myserver.local/jellyfin as if it is a directory on your existing website.

*Adjusting for your network values, of course.

## Conclusion

This solution should provide a secure and non-invasive method for proxying a Jellyfin server with little to no impact 
on an existing Apache2 configuration following the vanillia configs as an example.

Please feel free to implement this solution at your convenience should it suit your needs. Comments are welcome.

