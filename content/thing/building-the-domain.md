---
title: "Lab Domain Controller++" date: 2019-11-29 13:17:32-05:00 draft: false
tags: ["home lab"]
---

This describes my effort to configure a home samba 4 domain controller. It was
written mainly for my own reference during the process and later, when I
inevitably need to figure out WTF I did.

The starting point is a base installation of Debian 10 Buster. The goal is a
system which can:

- Act as a Domain Controller for our lab's Windows 10 hosts
- Serve authoritative DNS for our home network
- Provide DHCP on our lab network, and register provisioned clients with DNS
- Provide RADIUS authentication for our network
- Serve as Kubernetes master for our lab cluster

# Planning

The domain: home.funkhouse.rs DC hostname: pharoah.home.funkhouse.rs DC static
IP: 10.42.16.2/16

The existing network uses a `dnsmasq` DNS server at 10.42.0.2. This will be
deprecated after the domain is online. For the time, it will be the upstream DNS
resolver.

# Installing SAMBA 4

This section heavily references [the SAMBA Wiki
entry](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller)
for the same topic.

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

## Planning Domain Provisioning

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
have BIND9 installed before I continue provisioning. Let's do that now.

# Installing BIND9

I've already got BIND9 installed, apparently. I probably did this early on in
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

## Configuring it

The documentation [concerning BIND9
backends](https://wiki.samba.org/index.php/BIND9_DLZ_DNS_Back_End#Configuring_the_BIND9_DLZ_Module)
on the SAMBA Wiki describes how to configure BIND9 to load the DLZ module.
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

Looks like a problem; BIND9 is now looking for a file which doesn't exist. This
may be a tangled dependency issue, so I'm going to go back and provision the
SAMBA domain and point it at BIND9. I'm hoping that creates the required file.

# Provisioning the Domain

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

## Trying BIND9 Again

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
