---
title: "New Lab Infrastructure: Part 2"
description: In which FreeNAS is joined to the domain.
date: 2019-12-06T14:18:32-05:00
draft: false
tags: ["domain controller", "home lab", "freenas", "letsencrypt"]
---

This is part two of my effort to configure new lab infrastructure. It
was written mainly for my own reference during the process and later, when I
inevitably need to figure out WTF I did. Having my storage trust my IDP
is also nice.

root@magpie[~]# pkg search -g \*certbot\*
py27-certbot-0.39.0,1          Let's Encrypt client
py27-certbot-apache-0.39.0     Apache plugin for Certbot
py27-certbot-dns-cloudflare-0.39.0 Cloudflare DNS plugin for Certbot
py27-certbot-dns-cloudxns-0.39.0 CloudXNS DNS Authenticator plugin for Certbot
py27-certbot-dns-digitalocean-0.39.0 DigitalOcean DNS Authenticator plugin for Certbot
py27-certbot-dns-dnsimple-0.39.0 DNSimple DNS Authenticator plugin for Certbot
py27-certbot-dns-dnsmadeeasy-0.39.0 DNS Made Easy DNS Authenticator plugin for Certbot
py27-certbot-dns-gehirn-0.39.0 Gehirn Infrastructure Service DNS Authenticator plugin for Certbot
py27-certbot-dns-google-0.39.0 Google Cloud DNS Authenticator plugin for Certbot
py27-certbot-dns-linode-0.39.0 Linode DNS Authenticator plugin for Certbot
py27-certbot-dns-luadns-0.39.0 LuaDNS Authenticator plugin for Certbot
py27-certbot-dns-nsone-0.39.0  NS1 DNS Authenticator plugin for Certbot
py27-certbot-dns-ovh-0.39.0    OVH DNS Authenticator plugin for Certbot
py27-certbot-dns-rfc2136-0.39.0 RFC 2136 DNS Authenticator plugin for Certbot
py27-certbot-dns-route53-0.39.0 Route53 DNS Authenticator plugin for Certbot
py27-certbot-dns-sakuracloud-0.39.0 Sakura Cloud DNS Authenticator plugin for Certbot
py27-certbot-nginx-0.39.0      NGINX plugin for Certbot
py36-certbot-0.39.0,1          Let's Encrypt client
py36-certbot-apache-0.39.0     Apache plugin for Certbot
py36-certbot-dns-cloudflare-0.39.0 Cloudflare DNS plugin for Certbot
py36-certbot-dns-cloudxns-0.39.0 CloudXNS DNS Authenticator plugin for Certbot
py36-certbot-dns-digitalocean-0.39.0 DigitalOcean DNS Authenticator plugin for Certbot
py36-certbot-dns-dnsimple-0.39.0 DNSimple DNS Authenticator plugin for Certbot
py36-certbot-dns-dnsmadeeasy-0.39.0 DNS Made Easy DNS Authenticator plugin for Certbot
py36-certbot-dns-gehirn-0.39.0 Gehirn Infrastructure Service DNS Authenticator plugin for Certbot
py36-certbot-dns-google-0.39.0 Google Cloud DNS Authenticator plugin for Certbot
py36-certbot-dns-linode-0.39.0 Linode DNS Authenticator plugin for Certbot
py36-certbot-dns-luadns-0.39.0 LuaDNS Authenticator plugin for Certbot
py36-certbot-dns-nsone-0.39.0  NS1 DNS Authenticator plugin for Certbot
py36-certbot-dns-ovh-0.39.0    OVH DNS Authenticator plugin for Certbot
py36-certbot-dns-rfc2136-0.39.0 RFC 2136 DNS Authenticator plugin for Certbot
py36-certbot-dns-route53-0.39.0 Route53 DNS Authenticator plugin for Certbot
py36-certbot-dns-sakuracloud-0.39.0 Sakura Cloud DNS Authenticator plugin for Certbot
py36-certbot-nginx-0.39.0      NGINX plugin for Certbot
root@magpie[~]# pkg install py36-certbot py36-certbot-dns-google
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 20 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
        py36-certbot: 0.39.0,1 [FreeBSD]
        py36-certbot-dns-google: 0.39.0 [FreeBSD]
        py36-distro: 1.4.0_1 [FreeBSD]
        py36-josepy: 1.2.0 [FreeBSD]
        py36-acme: 0.39.0,1 [FreeBSD]
        py36-pyrfc3339: 1.1 [FreeBSD]
        py36-zope.interface: 4.6.0 [FreeBSD]
        py36-zope.component: 4.2.2 [FreeBSD]
        py36-zope.event: 4.1.0 [FreeBSD]
        py36-parsedatetime: 2.5 [FreeBSD]
        py36-configobj: 5.0.6_1 [FreeBSD]
        py36-configargparse: 0.15.1 [FreeBSD]
        py36-google-api-python-client: 1.7.6 [FreeBSD]
        py36-google-auth-httplib2: 0.0.3 [FreeBSD]
        py36-google-auth: 1.7.1 [FreeBSD]
        py36-rsa: 3.4.2_1 [FreeBSD]
        py36-cachetools: 2.0.0 [FreeBSD]
        py36-uritemplate: 3.0.0 [FreeBSD]
        py36-oauth2client: 4.1.3 [FreeBSD]
        py36-mock: 3.0.5 [FreeBSD]

Number of packages to be installed: 20

The process will require 11 MiB more space.
2 MiB to be downloaded.

Proceed with this action? [y/N]: Y

...


root@magpie[/etc]# mkdir -p /etc/letsencrypt
root@magpie[/etc]# ls -d /etc/letsencrypt
/etc/letsencrypt
root@magpie[/etc]# ls -ld /etc/letsencrypt
drwxr-xr-x  2 root  wheel  0 Dec  6 11:58 /etc/letsencrypt
root@magpie[/etc]# cd letsencrypt
root@magpie[/etc/letsencrypt]# mv /mnt/tank0/Home/HOME/christian/Documents/silent-bird-337-30532e1a8a92.json ./magpie-cert-renewer-credentials.json
mv: failed to set acl entries for ./magpie-cert-renewer-credentials.json: Operation not supported
root@magpie[/etc/letsencrypt]# ls
magpie-cert-renewer-credentials.json
root@magpie[/etc/letsencrypt]# ls -l
total 4
-rwxrwxr-x  1 HOME\christian  HOME\domain users  2338 Dec  6 11:53 magpie-cert-renewer-credentials.json
root@magpie[/etc/letsencrypt]# chown root:root ./magpie-cert-renewer-credentials.json
chown: root: illegal group name
root@magpie[/etc/letsencrypt]# chown root:wheel ./magpie-cert-renewer-credentials.json
root@magpie[/etc/letsencrypt]# chmod 400 ./magpie-cert-renewer-credentials.jsonroot@magpie[/etc/letsencrypt]# ll
total 12
drwxr-xr-x   2 root  wheel  -       64 Dec  6 11:59 ./
drwxr-xr-x  32 root  wheel  uarch 8256 Dec  6 11:58 ../
-r--------   1 root  wheel  -     2338 Dec  6 11:53 magpie-cert-renewer-credentials.json
root@magpie[/etc/letsencrypt]#

certbot certonly \
  --dns-google \
  --dns-google-credentials /etc/letsencrypt/magpie-cert-renewer-credentials.json \
  -d magpie.home.funkhouse.rs

root@magpie[/etc/letsencrypt]# certbot certonly \>   --dns-google \
>   --dns-google-credentials /etc/letsencrypt/magpie-cert-renewer-credentials.json \
>   -d magpie.home.funkhouse.rs
An unexpected error occurred:
pkg_resources.ContextualVersionConflict: (setuptools 39.0.1 (/usr/local/lib/pyth
on3.6/site-packages), Requirement.parse('setuptools>=40.3.0'), {'google-auth'})
Please see the logfile '/tmp/tmp2qr8z7y5/log' for more details.root@magpie[/etc/letsencrypt]#

:-/

root@magpie[/etc/letsencrypt]# pkg search -g \*google-auth\*py27-google-auth-1.7.1         Google Authentication Librarypy27-google-auth-httplib2-0.0.3 Google Authentication Library: httplib2 transport
py36-google-auth-1.7.1         Google Authentication Library
py36-google-auth-httplib2-0.0.3 Google Authentication Library: httplib2 transport
root@magpie[/etc/letsencrypt]# pkg install py36-google-auth
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Checking integrity... done (0 conflicting)
The most recent versions of packages are already installed
root@magpie[/etc/letsencrypt]# pkg install py27-google-auth-httplib2
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 8 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
        py27-google-auth-httplib2: 0.0.3 [FreeBSD]
        py27-httplib2: 0.14.0 [FreeBSD]
        py27-google-auth: 1.7.1 [FreeBSD]
        py27-rsa: 3.4.2_1 [FreeBSD]
        py27-pyasn1: 0.4.7 [FreeBSD]
        py27-six: 1.12.0 [FreeBSD]
        py27-pyasn1-modules: 0.2.7 [FreeBSD]
        py27-cachetools: 2.0.0 [FreeBSD]

Number of packages to be installed: 8

The process will require 4 MiB more space.
516 KiB to be downloaded.

Proceed with this action? [y/N]: Y
[1/8] Fetching py27-google-auth-httplib2-0.0.3.txz: 100%   10 KiB   9.9kB/s    00:01
[2/8] Fetching py27-httplib2-0.14.0.txz: 100%  103 KiB 105.5kB/s    00:01
[3/8] Fetching py27-google-auth-1.7.1.txz: 100%   68 KiB  69.9kB/s    00:01
[4/8] Fetching py27-rsa-3.4.2_1.txz: 100%   50 KiB  51.2kB/s    00:01
[5/8] Fetching py27-pyasn1-0.4.7.txz: 100%  101 KiB 103.5kB/s    00:01
[6/8] Fetching py27-six-1.12.0.txz: 100%   17 KiB  17.6kB/s    00:01
[7/8] Fetching py27-pyasn1-modules-0.2.7.txz:  36%   56 KiB  57.3kB/s    00:01 E[7/8] Fetching py27-pyasn1-modules-0.2.7.txz: 100%  152 KiB 155.6kB/s    00:01
[8/8] Fetching py27-cachetools-2.0.0.txz: 100%   15 KiB  15.2kB/s    00:01
Checking integrity... done (0 conflicting)
[1/8] Installing py27-pyasn1-0.4.7...
[1/8] Extracting py27-pyasn1-0.4.7: 100%
[2/8] Installing py27-rsa-3.4.2_1...
[2/8] Extracting py27-rsa-3.4.2_1: 100%
[3/8] Installing py27-six-1.12.0...
[3/8] Extracting py27-six-1.12.0: 100%
[4/8] Installing py27-pyasn1-modules-0.2.7...
[4/8] Extracting py27-pyasn1-modules-0.2.7: 100%
[5/8] Installing py27-cachetools-2.0.0...
[5/8] Extracting py27-cachetools-2.0.0: 100%
[6/8] Installing py27-httplib2-0.14.0...
[6/8] Extracting py27-httplib2-0.14.0: 100%
[7/8] Installing py27-google-auth-1.7.1...
[7/8] Extracting py27-google-auth-1.7.1: 100%
[8/8] Installing py27-google-auth-httplib2-0.0.3...
[8/8] Extracting py27-google-auth-httplib2-0.0.3: 100%
root@magpie[/etc/letsencrypt]#
root@magpie[/etc/letsencrypt]# pkg install py36-setuptools
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 1 package(s) will be affected (of 0 checked):

Installed packages to be UPGRADED:
        py36-setuptools: 39.0.1 -> 41.4.0 [FreeBSD]

Number of packages to be upgraded: 1

504 KiB to be downloaded.

Proceed with this action? [y/N]: Y
[1/1] Fetching py36-setuptools-41.4.0.txz: 100%  504 KiB 515.9kB/s    00:01
Checking integrity... done (0 conflicting)
[1/1] Upgrading py36-setuptools from 39.0.1 to 41.4.0...
[1/1] Extracting py36-setuptools-41.4.0: 100%
root@magpie[/etc/letsencrypt]# certbot certonly \
  --dns-google \
  --dns-google-credentials /etc/letsencrypt/magpie-cert-renewer-credentials.json \
  -d magpie.home.funkhouse.rs
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-google, Installer None
Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel): christian@funkhouse.rs

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(A)gree/(C)ancel: A

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about our work
encrypting the web, EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for magpie.home.funkhouse.rs
URL being requested: GET https://www.googleapis.com/discovery/v1/apis/dns/v1/rest
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones?dnsName=magpie.home.funkhouse.rs.&alt=json
Attempting refresh to obtain initial access_token
Refreshing access_token
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones?dnsName=home.funkhouse.rs.&alt=json
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones?dnsName=funkhouse.rs.&alt=json
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones/4240304521212323910/rrsets?alt=json
URL being requested: POST https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones/4240304521212323910/changes?alt=json
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones/4240304521212323910/changes/18?alt=json
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones/4240304521212323910/changes/18?alt=json
Waiting 60 seconds for DNS changes to propagate
Waiting for verification...
Cleaning up challenges
URL being requested: GET https://www.googleapis.com/discovery/v1/apis/dns/v1/rest
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones?dnsName=magpie.home.funkhouse.rs.&alt=json
Attempting refresh to obtain initial access_token
Refreshing access_token
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones?dnsName=home.funkhouse.rs.&alt=json
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones?dnsName=funkhouse.rs.&alt=json
URL being requested: GET https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones/4240304521212323910/rrsets?alt=json
URL being requested: POST https://dns.googleapis.com/dns/v1/projects/silent-bird-337/managedZones/4240304521212323910/changes?alt=json

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /usr/local/etc/letsencrypt/live/magpie.home.funkhouse.rs/fullchain.pem
   Your key file has been saved at:
   /usr/local/etc/letsencrypt/live/magpie.home.funkhouse.rs/privkey.pem
   Your cert will expire on 2020-03-05. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /usr/local/etc/letsencrypt. You should
   make a secure backup of this folder now. This configuration
   directory will also contain certificates and private keys obtained
   by Certbot so making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

root@magpie[/etc/letsencrypt]# ls -l /etc/certificates
total 8
drwxr-xr-x  2 root  wheel   192 Dec  1 12:35 CA
-rw-r--r--  1 root  wheel  2989 Dec  1 12:35 s3.crt
-r--------  1 root  wheel  1704 Dec  1 12:35 s3.key
root@magpie[/etc/letsencrypt]# ls /etc/letsencrypt
magpie-cert-renewer-credentials.json
root@magpie[/etc/letsencrypt]#
```

```sh
#!/bin/sh
# Redeploy the Let's Encrypt certificates, and then restart nginx.
# This is intended for use as a certbot post-hook.
/root/deploy-freenas/deploy_freenas.py \
  --config /root/deploy-freenas/deploy_config \
&& service nginx restart
```

```sh
certbot renew \
  --config-dir /root/letsencrypt \
  --post-hook /root/scripts/redeploy-nginx-certificates.sh
```
