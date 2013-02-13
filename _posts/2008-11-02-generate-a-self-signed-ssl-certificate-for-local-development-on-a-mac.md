---
layout: post
title: Generate a self-signed SSL certificate for local development on a Mac
lead: "First, make a home for the new SSL files—I use /etc/apache2/ssl.  Open up a ter­mi­nal win­dow, cd to the new direc­tory and issue the fol­low­ing com­mand to cre­ate a host key file."
---
### Generate a host key

First, make a home for the new SSL files&mdash;I use /etc/apache2/ssl. Open up
a terminal window, cd to the new directory and issue the following command to
create a host key file.

    sudo ssh-keygen -f host.key

### Generate a certificate request file

This command will create a certificate request file. A certificate request file
contains information about your organization that will be used in the SSL
certificate. The command will ask you a bunch of questions; because this is for
local development, nonsense will suffice.

    sudo openssl req -new -key host.key -out request.csr

### Create the SSL certificate

Create a self-signed SSL certificate using the request file.

    sudo openssl x509 -req -days 365 -in request.csr -signkey host.key -out server.crt

### Apache

Add the following to your Apache configuration to use the new certificate:

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/server.crt
    SSLCertificateKeyFile /etc/apache2/ssl/server.key

Restart Apache with `sudo apachectl restart` and try our your new certificate.
