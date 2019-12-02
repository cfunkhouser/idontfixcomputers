---
title: "New Lab Infrastructure: Part 1"
date: 2019-11-29 13:17:32-05:00
draft: false
tags: ["domain controller", "home lab", "samba"]
---

This describes my effort to configure new lab infrastructure. It was written
mainly for my own reference during the process and later, when I inevitably need
to figure out WTF I did. This part focuses on the Samba 4 Active Directory
Domain Controller.

## Planning

The eventual goal of this new infrastructure is:

- Active Directory-compatible Domain Controller for our lab's Windows 10 hosts
- Proper DNS, with reverse lookups
- DHCP on our lab network which registers provisioned clients with DNS
- RADIUS authentication for WiFi
- FreeNAS fully integrated with the domain, hosting all storage (SMB/CIFS,
  iSCSI, NFS)
- A Kubernetes cluster with integrated authentication

The details of the new network:

- The domain: `home.funkhouse.rs`
- DC hostname: `pharoah.home.funkhouse.rs`
- DC static IP: `10.42.16.2/16`
- The network: `10.42.0.0/16`

The existing network uses a `dnsmasq` DNS server at 10.42.0.2. This will be
deprecated after the domain is online. For the time, it will be the upstream DNS
resolver.

## Installing Samba 4

The starting point for this exercise is a fresh installation of Debian Buster on
the target hardware. I'll discuss the hardware elsewhere. It's not super
impressive, but it'll do for all this. I closely follow [the Samba Wiki
entry](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller)
for this exercise.

It's worth noting here that the document refers to `/usr/local/samba/private` in
many places, but this is not the location Debian Buster's packaged samba puts
things. Instead, everything lives at `/var/lib/samba/private`.

```console
$ sudo apt-get install \
    acl \
    attr \
    samba \
    samba-dsdb-modules \
    samba-vfs-modules \
    winbind \
    libpam-winbind \
    libnss-winbind \
    libpam-krb5 \
    krb5-config \
    krb5-user
```

During this installation, several Kerberos settings are chosen. Kerberos v5
realm name is `HOME.FUNKHOUSE.RS`. For all of the hostname settings, only the
host `pharoah.home.funkhouse.rs` is entered.

### Planning Domain Provisioning

Looking around briefly:

```console
christian@pharoah:~$ samba-tool --version
4.9.5-Debian
OK 29 Nov 2019 13:40:15 EST
christian@pharoah:~$ samba-tool domain provision --help
Usage: samba-tool domain provision [options]

Provision a domain.


Options:
  -h, --help            show this help message and exit
  --interactive         Ask for names
  --domain=DOMAIN       NetBIOS domain name to use
  --domain-guid=GUID    set domainguid (otherwise random)
  --domain-sid=SID      set domainsid (otherwise random)
  --ntds-guid=GUID      set NTDS object GUID (otherwise random)
  --invocationid=GUID   set invocationid (otherwise random)
  --host-name=HOSTNAME  set hostname
  --host-ip=IPADDRESS   set IPv4 ipaddress
  --host-ip6=IP6ADDRESS
                        set IPv6 ipaddress
  --site=SITENAME       set site name
  --adminpass=PASSWORD  choose admin password (otherwise random)
  --krbtgtpass=PASSWORD
                        choose krbtgt password (otherwise random)
  --dns-backend=NAMESERVER-BACKEND
                        The DNS server backend. SAMBA_INTERNAL is the builtin
                        name server (default), BIND9_FLATFILE uses bind9 text
                        database to store zone information, BIND9_DLZ uses
                        samba4 AD to store zone information, NONE skips the
                        DNS setup entirely (not recommended)
  --dnspass=PASSWORD    choose dns password (otherwise random)
  --root=USERNAME       choose 'root' unix username
  --nobody=USERNAME     choose 'nobody' user
  --users=GROUPNAME     choose 'users' group
  --blank               do not add users or groups, just the structure
  --server-role=ROLE    The server role (domain controller | dc | member
                        server | member | standalone). Default is dc.
  --function-level=FOR-FUN-LEVEL
                        The domain and forest function level (2000 | 2003 |
                        2008 | 2008_R2 - always native). Default is (Windows)
                        2008_R2 Native.
  --base-schema=BASE-SCHEMA
                        The base schema files to use. Default is (Windows)
                        2008_R2.
  --next-rid=NEXTRID    The initial nextRid value (only needed for upgrades).
                        Default is 1000.
  --partitions-only     Configure Samba's partitions, but do not modify them
                        (ie, join a BDC)
  --use-rfc2307         Use AD to store posix attributes (default = no)
  --machinepass=PASSWORD
                        choose machine password (otherwise random)
  --plaintext-secrets   Store secret/sensitive values as plain text on
                        disk(default is to encrypt secret/ensitive values)
  --backend-store=BACKENDSTORE
                        Specify the database backend to be used (default is
                        tdb)
  --targetdir=DIR       Set target directory (where to store provision)
  -q, --quiet           Be quiet

  Samba Common Options:
    -s FILE, --configfile=FILE
                        Configuration file
    -d DEBUGLEVEL, --debuglevel=DEBUGLEVEL
                        debug level
    --option=OPTION     set smb.conf option from command line
    --realm=REALM       set the realm name

  Version Options:
    -V, --version       Display version number
OK 29 Nov 2019 13:40:51 EST
```

Between this and the linked Wiki article, it looks like I need to make sure I
have BIND 9 installed before I continue provisioning. Let's do that now.

## Installing BIND 9

I've already got BIND 9 installed, apparently. I probably did this early on in
installation. Oh well.

```console
christian@pharoah:~$ sudo apt-get install bind9
Reading package lists... Done
Building dependency tree
Reading state information... Done
bind9 is already the newest version (1:9.11.5.P4+dfsg-5.1).
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
OK 29 Nov 2019 13:48:32 EST
$ bind9-config --version
VERSION=9.11.5-P4-5.1-Debian
OK 29 Nov 2019 13:49:15 EST
```

### Configuring it

The documentation [concerning BIND 9
backends](https://wiki.samba.org/index.php/BIND9_DLZ_DNS_Back_End#Configuring_the_BIND9_DLZ_Module)
on the Samba Wiki describes how to configure BIND 9 to load the DLZ module.
However, the location listed in the documentation does not exist on Buster.
Let's find it:

```console
christian@pharoah:~$ dpkg --get-selections |grep samba
python-samba                                    install
samba                                           install
samba-common                                    install
samba-common-bin                                install
samba-dsdb-modules:amd64                        install
samba-libs:amd64                                install
samba-vfs-modules:amd64                         install
OK 29 Nov 2019 13:54:59 EST
christian@pharoah:~$ dpkg-query -L samba-libs | grep dlz
/usr/lib/x86_64-linux-gnu/samba/bind9/dlz_bind9.so
/usr/lib/x86_64-linux-gnu/samba/bind9/dlz_bind9_10.so
/usr/lib/x86_64-linux-gnu/samba/bind9/dlz_bind9_11.so
/usr/lib/x86_64-linux-gnu/samba/bind9/dlz_bind9_9.so
OK 29 Nov 2019 13:55:10 EST
```

Looks like `/usr/lib/x86_64-linux-gnu/samba/bind9` is the magic path. Now we
create a `named.conf.dlz` in `/etc/bind`:

```text
#
# This configures dynamically loadable zones (DLZ) from AD schema
# Uncomment only single database line, depending on your BIND version
#
dlz "AD DNS Zone" {
    database "dlopen /usr/lib/x86_64-linux-gnu/samba/bind9/dlz_bind9_11.so";
};
```

Make sure that's included in `named.conf`:

```text
// This is the primary configuration file for the BIND DNS server named.
//
// Please read /usr/share/doc/bind9/README.Debian.gz for information on the
// structure of BIND configuration files in Debian, *BEFORE* you customize
// this configuration file.
//
// If you are just adding zones, please do that in /etc/bind/named.conf.local

include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";

// Include the samba DLZ configuration.
include "/etc/bind/named.conf.dlz";
```

Then restart `bind` and make sure it works:

```console
christian@pharoah:bind$ sudo service bind9 restart
Job for bind9.service failed because the control process exited with error code.
See "systemctl status bind9.service" and "journalctl -xe" for details.
FAIL(1) 29 Nov 2019 14:08:52 EST
christian@pharoah:bind$ sudo service bind9 status
● bind9.service - BIND Domain Name Server
   Loaded: loaded (/lib/systemd/system/bind9.service; enabled; vendor preset: enabled)
   Active: failed (Result: exit-code) since Fri 2019-11-29 14:08:52 EST; 5s ago
     Docs: man:named(8)
  Process: 18595 ExecStart=/usr/sbin/named $OPTIONS (code=exited, status=1/FAILURE)

Nov 29 14:08:51 pharoah named[18596]: Loading 'AD DNS Zone' using driver dlopen
Nov 29 14:08:52 pharoah named[18596]: samba_dlz: Failed to connect to Failed to connect to /var/lib/samba/private/dns/sam.ldb: Unable to open tdb
Nov 29 14:08:52 pharoah named[18596]: dlz_dlopen of 'AD DNS Zone' failed
Nov 29 14:08:52 pharoah named[18596]: SDLZ driver failed to load.
Nov 29 14:08:52 pharoah named[18596]: DLZ driver failed to load.
Nov 29 14:08:52 pharoah named[18596]: loading configuration: failure
Nov 29 14:08:52 pharoah named[18596]: exiting (due to fatal error)
Nov 29 14:08:52 pharoah systemd[1]: bind9.service: Control process exited, code=exited, status=1/FAILURE
Nov 29 14:08:52 pharoah systemd[1]: bind9.service: Failed with result 'exit-code'.
Nov 29 14:08:52 pharoah systemd[1]: Failed to start BIND Domain Name Server.
FAIL(3) 29 Nov 2019 14:09:26 EST
```

Fuck.

```console
christian@pharoah:bind$ ls -l /var/lib/samba/private/dns/
ls: cannot access '/var/lib/samba/private/dns/': No such file or directory
FAIL(2) 29 Nov 2019 14:11:27 EST
christian@pharoah:bind$ ls -l /var/lib/samba/private
total 848
drwx------ 2 root root   4096 Nov 29 14:09 msg.sock
-rw------- 1 root root   8888 Nov 29 13:39 netlogon_creds_cli.tdb
-rw------- 1 root root 421888 Nov 29 13:39 passdb.tdb
-rw------- 1 root root 430080 Nov 29 13:39 secrets.tdb
OK 29 Nov 2019 14:11:29 EST
```

Looks like a problem; BIND 9 is now looking for a file which doesn't exist. This
may be a tangled dependency issue, so I'm going to go back and provision the
Samba domain and point it at BIND 9. I'm hoping that creates the required file.

## Provisioning the Domain

Let's do it:

```console
christian@pharoah:bind$ sudo samba-tool domain provision \
>     --use-rfc2307 \
>     --realm HOME.FUNKHOUSE.RS \
>     --domain HOME \
>     --server-role dc \
>     --dns-backend BIND9_DLZ \
>     --interactive
Realm [HOME.FUNKHOUSE.RS]:
Domain [HOME]:
Server Role (dc, member, standalone) [dc]:
DNS backend (SAMBA_INTERNAL, BIND9_FLATFILE, BIND9_DLZ, NONE) [SAMBA_INTERNAL]:  BIND9_DLZ
Administrator password:
Retype password:
ERROR(<class 'samba.provision.ProvisioningError'>): Provision failed - ProvisioningError: guess_names: 'server role=standalone server' in /etc/sa
mba/smb.conf must match chosen server role 'active directory domain controller'!  Please remove the smb.conf file and let provision generate it
  File "/usr/lib/python2.7/dist-packages/samba/netcmd/domain.py", line 538, in run
    backend_store=backend_store)
  File "/usr/lib/python2.7/dist-packages/samba/provision/__init__.py", line 2187, in provision
    sitename=sitename, rootdn=rootdn, domain_names_forced=(samdb_fill == FILL_DRS))
  File "/usr/lib/python2.7/dist-packages/samba/provision/__init__.py", line 629, in guess_names
    raise ProvisioningError("guess_names: 'server role=%s' in %s must match chosen server role '%s'!  Please remove the smb.conf file and let pro
vision generate it" % (lp.get("server role"), lp.configfile, serverrole))
FAIL(255) 29 Nov 2019 14:16:57 EST
```

Fuck.

```console
christian@pharoah:bind$ sudo mv /etc/samba/smb.conf{,.orig}
OK 29 Nov 2019 14:18:11 EST
christian@pharoah:bind$ sudo samba-tool domain provision \
>      --use-rfc2307 \
>      --realm HOME.FUNKHOUSE.RS \
>      --domain HOME \
>      --server-role dc \
>      --dns-backend BIND9_DLZ \
>      --interactive
Realm [HOME.FUNKHOUSE.RS]:
Domain [HOME]:
Server Role (dc, member, standalone) [dc]:
DNS backend (SAMBA_INTERNAL, BIND9_FLATFILE, BIND9_DLZ, NONE) [SAMBA_INTERNAL]:  BIND9_DLZ
Administrator password:
Retype password:
Looking up IPv4 addresses
Looking up IPv6 addresses
No IPv6 address will be assigned
Setting up share.ldb
Setting up secrets.ldb
Setting up the registry
Setting up the privileges database
Setting up idmap db
Setting up SAM db
Setting up sam.ldb partitions and settings
Setting up sam.ldb rootDSE
Pre-loading the Samba 4 and AD schema
Unable to determine the DomainSID, can not enforce uniqueness constraint on local domainSIDs

Adding DomainDN: DC=home,DC=funkhouse,DC=rs
Adding configuration container
Setting up sam.ldb schema
Setting up sam.ldb configuration data
Setting up display specifiers
Modifying display specifiers and extended rights
Adding users container
Modifying users container
Adding computers container
Modifying computers container
Setting up sam.ldb data
Setting up well known security principals
Setting up sam.ldb users and groups
Setting up self join
Adding DNS accounts
Creating CN=MicrosoftDNS,CN=System,DC=home,DC=funkhouse,DC=rs
Creating DomainDnsZones and ForestDnsZones partitions
Populating DomainDnsZones and ForestDnsZones partitions
See /var/lib/samba/bind-dns/named.conf for an example configuration include file for BIND
and /var/lib/samba/bind-dns/named.txt for further documentation required for secure DNS updates
Setting up sam.ldb rootDSE marking as synchronized
Fixing provision GUIDs
A Kerberos configuration suitable for Samba AD has been generated at /var/lib/samba/private/krb5.conf
Merge the contents of this file with your system krb5.conf or replace it with this one. Do not create a symlink!
Setting up fake yp server settings
Once the above files are installed, your Samba AD server will be ready to use
Server Role:           active directory domain controller
Hostname:              pharoah
NetBIOS Domain:        HOME
DNS Domain:            home.funkhouse.rs
DOMAIN SID:            S-1-5-21-3415105275-643277267-3509453300
OK 29 Nov 2019 14:19:29 EST
```

Much better.

### Trying BIND 9 Again

```console
christian@pharoah:bind$ sudo service bind9 start
OK 29 Nov 2019 14:20:44 EST
christian@pharoah:bind$ sudo service bind9 status
● bind9.service - BIND Domain Name Server
   Loaded: loaded (/lib/systemd/system/bind9.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2019-11-29 14:20:44 EST; 2s ago
     Docs: man:named(8)
  Process: 19550 ExecStart=/usr/sbin/named $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 19551 (named)
    Tasks: 11 (limit: 4915)
   Memory: 42.4M
   CGroup: /system.slice/bind9.service
           └─19551 /usr/sbin/named -u bind

Nov 29 14:20:44 pharoah named[19551]: network unreachable resolving './DNSKEY/IN': 2001:500:9f::42#53
Nov 29 14:20:44 pharoah named[19551]: network unreachable resolving './NS/IN': 2001:500:9f::42#53
Nov 29 14:20:44 pharoah named[19551]: network unreachable resolving './DNSKEY/IN': 2001:503:ba3e::2:30#53
Nov 29 14:20:44 pharoah named[19551]: network unreachable resolving './NS/IN': 2001:503:ba3e::2:30#53
Nov 29 14:20:44 pharoah named[19551]: network unreachable resolving './DNSKEY/IN': 2001:dc3::35#53
Nov 29 14:20:44 pharoah named[19551]: network unreachable resolving './NS/IN': 2001:dc3::35#53
Nov 29 14:20:44 pharoah named[19551]: network unreachable resolving './DNSKEY/IN': 2001:500:1::53#53
Nov 29 14:20:44 pharoah named[19551]: network unreachable resolving './NS/IN': 2001:500:1::53#53
Nov 29 14:20:44 pharoah named[19551]: managed-keys-zone: Key 20326 for zone . acceptance timer complete: key now trusted
Nov 29 14:20:44 pharoah named[19551]: resolver priming query complete
OK 29 Nov 2019 14:20:47 EST
```

Hey, that's not outright failure! We'll need to deal with the IPv6 thing at some
point.

### Configuring System Resolver

For the DC to work correctly, it must act as its own resolver. Because of our
existing lab DNS infrastructure, we first must make sure BIND 9 uses the existing
DNS server as its upstream. We'll change this later, once everything else has
been pointed at this DC. We're going to configure it as a forwarding server, in
this case.

Make `/etc/bind/named.conf.options` look like this:

```text
options {
        directory "/var/cache/bind";

        // Forward to existing DNS server temporarily.
        // TODO(christian): Remove this when clients are moved to DC DNS.
        forwarders {
                10.42.0.2;
        };

        dnssec-validation auto;
        listen-on-v6 { any; };
};
```

Now, make sure `resolvconf` is not installed. `/etc/resolv.conf` needs to be
statically maintained.

```console
christian@pharoah:bind$ dpkg --get-selections | grep resolvconf
FAIL(1) 30 Nov 2019 12:14:02 EST
```

Great. Now, set `/etc/resolv.conf` to self-reference:

```text
search home.funkhouse.rs
nameserver 10.42.16.2
```

Reload BIND 9 to get it working, then do a quick sanity check:

```console
christian@pharoah:bind$ sudo systemctl reload bind9
OK 30 Nov 2019 12:19:50 EST
christian@pharoah:bind$ sudo systemctl status bind9
● bind9.service - BIND Domain Name Server
   Loaded: loaded (/lib/systemd/system/bind9.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2019-11-29 14:20:44 EST; 21h ago
     Docs: man:named(8)
  Process: 19550 ExecStart=/usr/sbin/named $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 18221 ExecReload=/usr/sbin/rndc reload (code=exited, status=0/SUCCESS)
 Main PID: 19551 (named)
    Tasks: 11 (limit: 4915)
   Memory: 49.0M
   CGroup: /system.slice/bind9.service
           └─19551 /usr/sbin/named -u bind

Nov 30 12:19:50 pharoah named[19551]: configuring command channel from '/etc/bind/rndc.key'
Nov 30 12:19:50 pharoah named[19551]: configuring command channel from '/etc/bind/rndc.key'
Nov 30 12:19:50 pharoah named[19551]: zone home.funkhouse.rs/NONE: (other) removed
Nov 30 12:19:50 pharoah named[19551]: zone _msdcs.home.funkhouse.rs/NONE: (other) removed
Nov 30 12:19:50 pharoah named[19551]: reloading configuration succeeded
Nov 30 12:19:50 pharoah named[19551]: reloading zones succeeded
Nov 30 12:19:50 pharoah named[19551]: samba_dlz: shutting down
Nov 30 12:19:50 pharoah named[19551]: all zones loaded
Nov 30 12:19:50 pharoah named[19551]: running
Nov 30 12:19:51 pharoah named[19551]: managed-keys-zone: Key 20326 for zone . acceptance timer complete: key now trusted
OK 30 Nov 2019 12:19:53 EST
christian@pharoah:bind$ dig heroku.com

; <<>> DiG 9.11.5-P4-5.1-Debian <<>> heroku.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1182
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 13, ADDITIONAL: 27

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: aaa3f054f7802eb8c8dda39a5de2a5b499a0b08c63af4ef6 (good)
;; QUESTION SECTION:
;heroku.com.                    IN      A

;; ANSWER SECTION:
heroku.com.             29      IN      A       50.19.85.132
heroku.com.             29      IN      A       50.19.85.156
heroku.com.             29      IN      A       50.19.85.154

;; AUTHORITY SECTION:

... truncated ...

;; Query time: 110 msec
;; SERVER: 10.42.16.2#53(10.42.16.2)
;; WHEN: Sat Nov 30 12:24:04 EST 2019
;; MSG SIZE  rcvd: 911

OK 30 Nov 2019 12:24:04 EST
christian@pharoah:bind$ host pharoah
pharoah.home.funkhouse.rs has address 10.42.16.2
OK 30 Nov 2019 12:29:03 EST
christian@pharoah:bind$ host pharoah.home.funkhouse.rs
pharoah.home.funkhouse.rs has address 10.42.16.2
OK 30 Nov 2019 12:29:10 EST
```

Fantastic.

### Fun with Reverse Zones

Now we get a chance to see if our BIND 9 / Samba integration is working by
creating a reverse zone for our subnet. Our network is `10.42.0.0/16`, so we'll
create a whole `/16` reverse zone:

```console
christian@pharoah:bind$ sudo samba-tool dns zonecreate pharoah.home.funkhouse.rs 42.10.in-addr.arpa
Failed to connect host 127.0.1.1 on port 135 - NT_STATUS_CONNECTION_REFUSED
Failed to connect host 127.0.1.1 (pharoah.home.funkhouse.rs) on port 135 - NT_STATUS_CONNECTION_REFUSED.
ERROR: Connecting to DNS RPC server pharoah.home.funkhouse.rs failed with (3221226038, 'The transport-connection attempt was refused by the remote system.')
FAIL(255) 30 Nov 2019 12:41:19 EST
```

Hrm. Looks like samba has to be running for this to succeed. Not sure why the
Wiki puts this step at this point. Guess I'll try again later.

## Kerberos

Samba (and Active Directory) use Kerberos for authentication of things. Now, we
copy the provisioned Kerberos configuration to the appropriate location.

```console
christian@pharoah:bind$ sudo mv /etc/krb5.conf{,.orig}
OK 30 Nov 2019 12:52:12 EST
christian@pharoah:bind$ sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
OK 30 Nov 2019 12:52:28 EST
```

## Starting Samba

At this point, I was having lots of weird behavior from various parts of the
samba stack. I restarted the DC, and things got much better. This revealed the
annoying fact that samba doesn't start when the system reboots:

```console
christian@pharoah:~$ ps ax | egrep "samba|smbd|nmbd|winbindd"
  750 pts/1    S+     0:00 grep -E samba|smbd|nmbd|winbindd
OK 30 Nov 2019 13:55:29 EST
```

The `samba-ad-dc.service` that ships with Debian Buster is masked out of the
box, so we need to:

1. Mask and disable the individual `smbd`, `nmbd`, and `winbind` services
2. Unmask and enable the `samba-ad-dc` service

```console
christian@pharoah:~$ sudo systemctl mask smbd nmbd winbind
[sudo] password for christian:
Created symlink /etc/systemd/system/smbd.service → /dev/null.
Created symlink /etc/systemd/system/nmbd.service → /dev/null.
Created symlink /etc/systemd/system/winbind.service → /dev/null.
OK 30 Nov 2019 13:58:45 EST
christian@pharoah:~$ sudo systemctl disable smbd nmbd winbind
Synchronizing state of smbd.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable smbd
Synchronizing state of nmbd.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable nmbd
Synchronizing state of winbind.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable winbind
Unit /etc/systemd/system/smbd.service is masked, ignoring.
Unit /etc/systemd/system/nmbd.service is masked, ignoring.
Unit /etc/systemd/system/winbind.service is masked, ignoring.
OK 30 Nov 2019 13:58:52 EST
christian@pharoah:~$ sudo systemctl unmask samba-ad-dc
Removed /etc/systemd/system/samba-ad-dc.service.
OK 30 Nov 2019 14:08:55 EST
christian@pharoah:system$ sudo systemctl enable samba-ad-dc.service
Synchronizing state of samba-ad-dc.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable samba-ad-dc
Created symlink /etc/systemd/system/multi-user.target.wants/samba-ad-dc.service → /lib/systemd/system/samba-ad-dc.service.
```

Then verify that everything worked:

```console
christian@pharoah:system$ sudo systemctl start samba-ad-dc.service
OK 30 Nov 2019 14:15:44 EST
christian@pharoah:system$ sudo systemctl status samba-ad-dc.service
● samba-ad-dc.service - Samba AD Daemon
   Loaded: loaded (/lib/systemd/system/samba-ad-dc.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2019-11-30 14:15:44 EST; 9s ago
     Docs: man:samba(8)
           man:samba(7)
           man:smb.conf(5)
 Main PID: 2563 (samba)
   Status: "smbd: ready to serve connections..."
    Tasks: 20 (limit: 4915)
   Memory: 138.6M
   CGroup: /system.slice/samba-ad-dc.service
           ├─2563 samba: root process
           ├─2564 samba: task[s3fs_parent]
           ├─2565 samba: task[dcesrv]
           ├─2566 samba: tfork waiter process
           ├─2567 samba: task[nbtd]
           ├─2568 samba: task[wrepl]
           ├─2569 /usr/sbin/smbd -D --option=server role check:inhibit=yes --foreground
           ├─2570 samba: task[ldapsrv]
           ├─2571 samba: task[cldapd]
           ├─2572 samba: task[kdc]
           ├─2573 samba: task[dreplsrv]
           ├─2574 samba: task[winbindd_parent]
           ├─2575 samba: task[ntp_signd]
           ├─2576 samba: tfork waiter process
           ├─2577 samba: task[kccsrv]
           ├─2578 /usr/sbin/winbindd -D --option=server role check:inhibit=yes --foreground
           ├─2579 samba: task[dnsupdate]
           ├─2587 /usr/sbin/smbd -D --option=server role check:inhibit=yes --foreground
           ├─2588 /usr/sbin/smbd -D --option=server role check:inhibit=yes --foreground
           └─2590 /usr/sbin/smbd -D --option=server role check:inhibit=yes --foreground

Nov 30 14:15:43 pharoah samba[2563]: root process[2563]:   Copyright Andrew Tridgell and the Samba Team 1992-2018
Nov 30 14:15:43 pharoah samba[2563]: root process[2563]: [2019/11/30 14:15:43.837965,  0] ../source4/smbd/server.c:773(binary_smbd_main)
Nov 30 14:15:43 pharoah samba[2563]: root process[2563]:   binary_smbd_main: samba: using 'standard' process model
Nov 30 14:15:44 pharoah winbindd[2578]: [2019/11/30 14:15:44.089636,  0] ../source3/winbindd/winbindd_cache.c:3160(initialize_winbindd_cache)
Nov 30 14:15:44 pharoah winbindd[2578]:   initialize_winbindd_cache: clearing cache and re-creating with version number 2
Nov 30 14:15:44 pharoah winbindd[2578]: [2019/11/30 14:15:44.098633,  0] ../lib/util/become_daemon.c:138(daemon_ready)
Nov 30 14:15:44 pharoah winbindd[2578]:   daemon_ready: STATUS=daemon 'winbindd' finished starting up and ready to serve connections
Nov 30 14:15:44 pharoah systemd[1]: Started Samba AD Daemon.
Nov 30 14:15:44 pharoah smbd[2569]: [2019/11/30 14:15:44.114847,  0] ../lib/util/become_daemon.c:138(daemon_ready)
Nov 30 14:15:44 pharoah smbd[2569]:   daemon_ready: STATUS=daemon 'smbd' finished starting up and ready to serve connections
OK 30 Nov 2019 14:15:53 EST
```

Noice.

### Authenticating

Let's test by logging in with the Administrator.

```
christian@pharoah:system$ kinit administrator
Password for administrator@HOME.FUNKHOUSE.RS: 
Warning: Your password will expire in 41 days on Sat 11 Jan 2020 01:40:24 PM EST
OK 30 Nov 2019 14:19:49 EST
```

I had changed the password for `HOME\Administrator` before I did this, but did
not capture the output. I used `sudo samba-tool user setpassword Administrator`
to do it.

Also, gonna have to do something about that 41 days shenanigans. That won't do.

### Fun with Reverse Zones Redux

Now that I can be sure that all of the appropriate daemons are running, and I
can log in as `HOME\Administrator`, I should be able to set that reverse zone:

```console
christian@pharoah:system$ samba-tool dns zonecreate pharoah.home.funkhouse.rs 42.10.in-addr.arpa
Zone 42.10.in-addr.arpa created successfully
OK 30 Nov 2019 14:22:40 EST
```

## Testing Samba

Run through some sanity checks that the Wiki recommends. First, the file shares:

```console
christian@pharoah:system$ sudo smbclient -L localhost -U christian
Enter HOME\christian's password:

        Sharename       Type      Comment
        ---------       ----      -------
        netlogon        Disk
        sysvol          Disk
        IPC$            IPC       IPC Service (Samba 4.9.5-Debian)
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            PHAROAH
OK 30 Nov 2019 16:06:33 EST
christian@pharoah:system$ smbclient //localhost/netlogon -Uchristian -c 'ls'
Unable to initialize messaging context
Enter HOME\christian's password:
  .                                   D        0  Fri Nov 29 14:19:23 2019
  ..                                  D        0  Fri Nov 29 14:19:26 2019

                948004776 blocks of size 1024. 897420696 blocks available
OK 30 Nov 2019 16:07:13 EST
```

That annoying `Unable to initialize messaging context` error is a known bug that
was fixed upstream: https://bugzilla.samba.org/show_bug.cgi?id=13925

Some `SRV` and `A` record verification:

```console
OK 30 Nov 2019 16:07:13 EST
christian@pharoah:system$ host -t SRV _ldap._tcp.home.funkhouse.rs
_ldap._tcp.home.funkhouse.rs has SRV record 0 100 389 pharoah.home.funkhouse.rs.
OK 30 Nov 2019 16:08:45 EST
christian@pharoah:system$ host -t SRV _kerberos._udp.home.funkhouse.rs
_kerberos._udp.home.funkhouse.rs has SRV record 0 100 88 pharoah.home.funkhouse.rs.
OK 30 Nov 2019 16:08:57 EST
christian@pharoah:system$ host -t A pharoah.home.funkhouse.rs
pharoah.home.funkhouse.rs has address 10.42.16.2
OK 30 Nov 2019 16:09:12 EST
```

Seems like all of that makes sense.

## Postscript

The install went relatively smoothly, to be honest. A few hiccups revolving
around operations orders, and some strangeness about how Debian packages Samba.
None of this was a show-stopper.

Next up: Configuring FreeNAS to work with this DC, so shares can live somewhere
provisioned for actual storage.
