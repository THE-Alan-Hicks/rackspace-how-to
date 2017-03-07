---
permalink: server-name-indication/
audit_date:
title: SNI - Server Name Indication
type: page
created_date: '2017-03-03'
created_by: Alan Hicks
last_modified_date: '2017-03-07'
last_modified_by: Alan Hicks
product: undefined
product_url: undefined
---

### Overview

SNI is an extension to the TLS/SSL protocols that allows a web server
to serve multiple encrypted websites from a single IP address in a
manner similar to unencrypted websites. Before the advent of SNI, every
encrypted website required a unique IP address. SNI can greatly
simplify the configuration of your webservers as well as your DNS zone
files. Since additional IP addresses are no longer required for TLS/SSL
websites, it also reduces your network complexity and can save you
money.

Throughout this document, we'll be utilizing examples with the Apache
httpd webserver, but the principles apply equally to other web services
such as nginx and Microsoft's IIS.

### How HTTP (Unencrypted) VirtualHosts Are Processed

For many years, web servers have served multiple web pages from a
single IP address using a facility known as Virtual Hosting. This makes
use of HTTP 1.1's Host header to inform the web server what hostname
the browser is requesting. A typical request might look something like
this:

    GET / HTTP/1.1
    Host: www.example.com
    User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:45.0) Gecko/20100101 Firefox/45.0
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    Accept-Language: en-US,en;q=0.5
    Accept-Encoding: gzip, deflate
    DNT: 1
    Connection: keep-alive

Here, we can clearly see that the Host header has specified
"www.example.com". Before retrieving the requested content, the web
server can consult its list of Virtual Hosts and determine which
content to serve based on this information. This is a simple approach
which has worked remarkably well for more than twenty years.

Here's a pair of traditional virtual hosts for example.com and
pattern.com. Both are listening on the same IP address and utilize
different document roots.

    <VirtualHost 192.168.1.1:80>
      ServerName  example.com
      ServerAlias www.example.com
      DocumentRoot /var/www/vhosts/example.com/
      <Directory /var/www/vhosts/example.com>
        Require all granted
      </Directory>
    </VirtualHost>
    <VirtualHost 192.168.1.1:80>
      ServerName  pattern.com
      ServerAlias www.pattern.com
      DocumentRoot /var/www/vhosts/pattern.com/
      <Directory /var/www/vhosts/pattern.com>
        Require all granted
      </Directory>
    </VirtualHost>

When a request comes into the IP address 192.168.1.1 on TCP port 80,
Apache receives the request and compares that value of the Host header
to ServerName and ServerAlias for each of them. When it finds a match,
it retrieves the content related to that VirtualHost. Because Apache
does not need to make any decisions about what content to serve until
after the Host header has been received, this works well.

### How SNI Differs From Traditional VirtualHosts

Website encryption complicates virtual hosting, primarily because
different web sites require different TLS/SSL certificates and keys.
This means that we can no longer rely exclusively upon the Host header.
Let's let's take a look at how you might configure HTTPS virtual hosts.

    <VirtualHost 192.168.1.1:443>
      ServerName  example.com
      ServerAlias www.example.com
      SSLEngine on
      SSLCertificateFile     /etc/pki/certs/example.com.crt
      SSLCertificateKeyFile  /etc/pki/private/example.com.key
      DocumentRoot /var/www/vhosts/example.com/
      <Directory /var/www/vhosts/example.com>
        Require all granted
      </Directory>
    </VirtualHost>

    <VirtualHost 192.168.1.1:443>
      ServerName  pattern.com
      ServerAlias www.pattern.com
      SSLEngine on
      SSLCertificateFile     /etc/pki/certs/pattern.com.crt
      SSLCertificateKeyFile  /etc/pki/private/pattern.com.key
      DocumentRoot /var/www/vhosts/pattern.com/
      <Directory /var/www/vhosts/pattern.com>
        Require all granted
      </Directory>
    </VirtualHost>

This example doesn't differ significantly from the previous one. The
predominant differences are the presence of various SSL options.
Because TLS/SSL certificates are each only valid for a small number of
domain names (typically one or two such as example.com and
www.example.com), we must manage and serve different certificates for
each site.

Apache accomplishes this by reviewing the data provided by the web
browser during the TLS/SSL handshake. As part of that handshake, a
browser which supports SNI will include information such as the
following:

    Extension: server_name
      Type: server_name (0x0000)
      Length: 19
      Server Name Indication extension
        Server Name list length: 17
        Server Name Type: host_name (0)
        Server Name length: 14
        Server Name: www.example.com

When Apache receives this request, it compares the value of the Server
Name field (www.example.com) against its VirtualHost contexts. When it
finds a match, it selects the appropriate TLS/SSL certificate based on
the value of SSLCertificateFile. Now that it knows which certificate is
valid for the requested hostname, it completes the handshake. This
ensures that the correct certificate and key are used, preventing
certificate mismatch errors.

### SNI Requirements

Historically, SNI adoption has been slow because it requires that both
the server and the client's browser support it. Since website owners
have no control over the browsers their customers use, they've been
reluctant to make such moves. That has finally changed.  Today,
virtually all systems support SNI, so the barriers to adoption have
vanished.

#### Web Server Requirements

    * Must use OpenSSL 0.9.8f or any later version
    * OpenSSL must have been built with the TLS Extensions option
    * The web service must have been compiled against this OpenSSL version

These requirements are met by Red Hat versions 6 and later, and all
versions of Ubuntu that Rackspace supports. If you are still running
Red Hat version 5, you should migrate to a supported version of RHEL as
soon as possible. (RHEL 5.x is no longer supported by Red Hat as of
March 31, 2017.)

#### Web Browser Requirements

    * All versions of Google Chrome
    * Mozilla Firefox version 2.0 or later
    * Internet Explorer 7 or later (not on Windows XP)

From the above requirements list, you can see that the only web
browsers and operating systems that do not support SNI are no longer
supported by anyone. Windows XP hasn't been supported by Microsoft
since April 8, 2014 and major browsers have supported SNI for more than
a decade. A complete list of SNI-aware browsers is maintained by
Wikipedia.[1](#1)

### References

1.  [Server Name Indication - Wikipedia](https://en.wikipedia.org/wiki/Server_Name_Indication#Support) <a name="1"></a>




