# Automatically Generate Rulesets for Apache mod_rewrite or Nginx for Intelligent HTTP C2 Redirection

![Python application](https://github.com/threatexpress/cs2modrewrite/workflows/Python%20application/badge.svg)
This project converts a Cobalt Strike profile to a functional mod_rewrite `.htaccess` or Nginx config file to support HTTP reverse proxy redirection to a Cobalt Strike teamserver. The use of reverse proxies provides protection to backend C2 servers from profiling, investigation, and general internet background radiation.

***Note***: You should test and tune the output as needed before deploying, but these scripts should handle the heavy lifting.

## Features

- Now requires Python 3.0+
- Supports the Cobalt Strike custom URI features as of CS 4.0
- Rewrite Rules based on valid C2 URIs (HTTP GET, POST, and Stager) and specified User-Agent string.
  - Result: Only requests to valid C2 endpoints with a specified UA string will be proxied to the teamserver by default.
- Uses a custom Malleable C2 profile to build a .htaccess file with corresponding mod_rewrite rules
- Uses a custom Malleable C2 profile to build a Nginx config with corresponding proxy_pass rules
- HTTP or HTTPS proxying to the Cobalt Strike teamserver
- HTTP 302 Redirection to a Legitimate Site for Non-Matching Requests

## Fork note

Please do not look at my ugly regex code for adding havoc profile support. Thanks.
```
python3 havoc2nginx.py -i /tmp/profile.yaotl -c http://127.0.0.1:30001 -r http://internet-server.lan -H redirc2
```

## Quick start

The havex.profile example is included for a quick test.

1) Run the script against a profile
2) Save the output to `.htaccess` or `/etc/nginx/nginx.conf` on your redirector
3) Modify as needed
4) Reload\restart the web server

## Apache mod_rewrite Example Usage using a remote include file

```
python3 cs2modrewrite.py -i havex.profile -c https://TEAMSERVER -r https://GOHERE -o /etc/apache2/redirect.rules
```

Example Apache Config

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    RemoteIPHeader X-Forwarded-For

    ErrorLog /var/log/apache2/redirector_error.log
    CustomLog /var/log/apache2/redirector_access.log combined
    ErrorDocument 401 " "
    ErrorDocument 403 " "
    ErrorDocument 404 " "
    ErrorDocument 500 " "
    ErrorDocument 503 " "

    # Include redirect.rules
    Include /etc/apache2/redirect.rules
</VirtualHost>
```

Consider Updating Apache Server Header, ServerTokens, and logging with something like the following.

```
## Update Apached Server Header, ServerTokens, and logging
echo "Update Update Apached Server Header, ServerTokens, and logging"
sed -i -e 's/\(ServerTokens\s\+\)OS/\1Prod/g' /etc/apache2/conf-enabled/security.conf
sed -i -e 's/\(ServerSignature\s\+\)On/\1Off/g' /etc/apache2/conf-enabled/security.conf
echo "SecServerSignature Server" >> /etc/apache2/conf-enabled/security.conf
echo "LogLevel alert rewrite:trace2" >> /etc/apache2/conf-enabled/security.conf

## Update Apached remoteip.conf
echo "Update Apached remoteip.conf"
echo "RemoteIPHeader X-Forwarded-For" >> /etc/apache2/conf-enabled/remoteip.conf

## Restart apache server
echo "Restart apache server"
systemctl restart apache2
```

## Apache mod_rewrite Example Usage using a .htaccess file

```
python3 cs2modrewrite.py -i havex.profile -c https://TEAMSERVER -r https://GOHERE -o /var/www/html/.htaccess
```

## Apache Rewrite Setup and Tips

### Enable Rewrite and Proxy

    apt-get install apache2
    a2enmod rewrite headers proxy proxy_http ssl cache
    a2dismod -f deflate
    service apache2 reload

**Note:** https://bluescreenofjeff.com/2016-06-28-cobalt-strike-http-c2-redirectors-with-apache-mod_rewrite/
"e0x70i pointed out in the comments below that if your Cobalt Strike Malleable C2 profile contains an Accept-Encoding header for gzip, your Apache install may compress that traffic by default and cause your Beacon to be unresponsive or function incorrectly. To overcome this, disable mod_deflate (via a2dismod deflate and add the No Encode ([NE]) flag to your rewrite rules. (Thank you, e0x70i!)"

### Enable SSL support

Ensure the following entries are in the site's config (i.e. `/etc/apache2/available-sites/*.conf`)

    # Enable SSL
    SSLEngine On
    # Enable SSL Proxy
    SSLProxyEngine On
    # Trust Self-Signed Certificates generated by CobaltStrike
    SSLProxyVerify none
    SSLProxyCheckPeerCN off
    SSLProxyCheckPeerName off
    SSLProxyCheckPeerExpire off
### .HTACCESS

If you plan on using mod_rewrite in .htaccess files (instead of the site's config file), you also need to enable the use of `.htaccess` files by changing `AllowOverride None` to `AllowOverride All`. For all websites, edit `/etc/apache2/apache.conf`

    <Directory /var/www/>
        Options FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>

Finally, restart apache once more for good measure.

`service apache2 restart`

### Troubleshooting

If you need to troubleshoot redirection rule behavior, enable detailed error tracing in your site's configuration file by adding the following line.

`LogLevel alert rewrite:trace5`

Next, reload apache, and monitor `/var/log/access.log` `/var/log/error.log` to see which rules are matching.

----------------------------------------------

## Nginx Example Usage

### Install Nginx

    apt-get install nginx nginx-extras

**Note:** `nginx-extras` is needed for custom server headers. If you can't get this package, then comment out the server header line in the resulting configuration file.

### Create Redirection Rules

Save the cs2nginx.py output to `/etc/nginx/nginx.conf` and modify as needed (SSL parameters).

`python3 ./cs2nginx.py -i havex.profile -c https://127.0.0.1 -r https://www.google.com -H mydomain.local >/etc/nginx/nginx.conf`

Finally, restart nginx after modifying the server configuration file.

`service nginx restart`

## Final Thoughts

Once redirection is configured and functioning, ensure your C2 servers only allow ingress from the redirector and your trusted IPs (VPN, office ranges, etc).

Consider adding additional redirector protections using GeoIP restrictions (mod_maxmind) and blacklists of bad user agents and IP ranges. Thanks to [@curi0usJack](https://twitter.com/curi0usJack) for the ideas.

## References

- [Joe Vest and Andrew Chiles - cs2modrewrite.py blog post](https://posts.specterops.io/automating-apache-mod-rewrite-and-cobalt-strike-malleable-c2-profiles-d45266ca642)

- [@bluescreenofjeff - Cobalt Strike HTTP C2 Redirectors with Apache mod_rewrite](https://bluescreenofjeff.com/2016-06-28-cobalt-strike-http-c2-redirectors-with-apache-mod_rewrite/)

- [Adam Brown - Resilient Red Team HTTPS Redirection Using Nginx](https://coffeegist.com/security/resilient-red-team-https-redirection-using-nginx/)

- [Apache - Apache mod_rewrite Documentation](http://httpd.apache.org/docs/current/mod/mod_rewrite.html)
