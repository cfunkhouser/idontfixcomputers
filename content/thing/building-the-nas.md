---
title: "Home-built NAS (deprecated)"
description: In which is incompletely described the method for developing a very
  old NAS.
date: 2016-07-25T14:53:45-04:00
draft: false
tags: ["nas", "home lab"]
---

Note: This document was never finished. I'm also no longer using this NAS.

## Install Debian 8

-   Installed Debian 8.3 netinst via USB, made by UNetBootin
-   Upgraded to Debain 8.4

Carved up the 120GB SSD into 110GB `/root` using `ext4` and 5GB swap. Using `ext4`
for the root partition because I don't want to boot from a non-Linux-native FS for reasons.

## Install ZFS for Linux

Followed instructions from https://github.com/zfsonlinux/zfs/wiki/Debian

`TODO(christian): Fill this in.`

## Configuring ZFS

### Preparing Disks

Listing all disks:

```shell
root@magpie:~# ll /dev/sd?
brw-rw---- 1 root disk 8,  0 Jun  1 17:01 /dev/sda
brw-rw---- 1 root disk 8, 16 Jun  1 18:05 /dev/sdb
brw-rw---- 1 root disk 8, 32 Jun  1 18:05 /dev/sdc
brw-rw---- 1 root disk 8, 48 Jun  1 18:06 /dev/sdd
brw-rw---- 1 root disk 8, 64 Jun  1 18:06 /dev/sde
```

/dev/sda contains the root FS, so leave that one out. It's recommended that ZFS
get your whole drive, instead of a single partition, so:

```shell
# for disk in /dev/sd{b,c,d,e} ; do parted -s ${disk} mklabel gpt ; done
```

This creates an empty GPT label on each device.

### Creating the `zpool`

I want some redundancy in my storage soltution, but because the idea is to
backup the entire filesystem using [CrashPlan](http://www.crashplan.com/), I
feel comfortable taking the two-failing-disks-at-once risk and having only one
parity disk. Therefor, I'm creating the new pool with RAIDZ (as opposed to
RAIDZ2, or higher).

I'm choosing the name `tank0` for my new zpool. `tank` as is tradition with ZFS,
and `0` because my NAS has 8 bays, and I'd like to eventually add a second pool.

```shell
# zpool create tank0 raidz /dev/sd{b,c,d,e}
```

Silent return after a few seconds is what you want. Confirm it worked by:

```shell
root@magpie:~# zpool list
NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
tank0  3.62T   114K  3.62T         -     0%     0%  1.00x  ONLINE  -
```

### Creating and mounting a dataset
I'm creating a basic dataset in `tank0` called `data0` because I'm wicked creative. This dataset has no explicit size, and so is free to use as much as the pool as it needs. I will invariably add additional datasets to the pool in the future, and will use the same scheme. Trying to predict the usage ahead of time never works, and I'd rather just let them all use what they need and let ZFS sort it out.

```shell
root@magpie:~# zpool create tank0/data0
```

I want the the filesystem for data mounted outside of the traditional filesystem. I've chosen `/data` as the path to store it.

```shell
root@magpie:~# zfs set mountpoint=/data/tank0 tank0
root@magpie:~# zfs get mountpoint tank0/data0
NAME         PROPERTY    VALUE              SOURCE
tank0/data0  mountpoint  /data/tank0/data0  inherited from tank0
```

## Docker

I want the ability to run various things on the NAS (media servers, backup software, etc) but don't want to let it have free reign on the system. To accomplish this, I'm going to use Docker instead of native Linux containers.

### Installing

Looks like it's included in Debian Jessie's repositories:

```shell
root@magpie:~# apt-cache search docker
karbon - vector graphics application for the Calligra Suite
docker - System tray for KDE3/GNOME2 docklet applications
kdocker - lets you dock any application into the system tray
python-docker - Python wrapper to access docker.io's control socket
python3-docker - Python 3 wrapper to access docker.io's control socket
ruby-docker-api - Ruby gem to interact with docker.io remote API
root@magpie:~# apt-cache show docker |grep Version
Version: 1.5-1
```

Ew gross, it's old. Luckily, it looks like [Docker provides an `apt` repository](https://docs.docker.com/engine/installation/linux/debian/), so I'll use that.

First, installing the key:

```shell
root@magpie:~# apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
... (verbose crap omitted)
gpg: requesting key 2C52609D from hkp server p80.pool.sks-keyservers.net
gpg: key 2C52609D: public key "Docker Release Tool (releasedocker) <docker@docker.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
```

Then, add the repository:

```shell
root@magpie:~# echo "deb https://apt.dockerproject.org/repo debian-jessie main" > /etc/apt/sources.list.d/docker.list
root@magpie:~# apt-get update
... (verbose crap omitted) ...
root@magpie:~# apt-cache show docker-engine |grep Version
Version: 1.11.2-0~jessie
Version: 1.11.1-0~jessie
Version: 1.11.0-0~jessie
Version: 1.10.3-0~jessie
Version: 1.10.2-0~jessie
Version: 1.10.1-0~jessie
Version: 1.10.0-0~jessie
Version: 1.9.1-0~jessie
Version: 1.9.0-0~jessie
Version: 1.8.3-0~jessie
Version: 1.8.2-0~jessie
Version: 1.8.1-0~jessie
Version: 1.8.0-0~jessie
Version: 1.7.1-0~jessie
Version: 1.7.0-0~jessie
Version: 1.6.2-0~jessie
Version: 1.6.1-0~jessie
Version: 1.6.0-0~jessie
Version: 1.5.0-0~jessie
```

Well, okay then. Guess I want the latest?

```shell
root@magpie:~# apt-get install docker-engine
... (verbose crap omitted)
root@magpie:~# service docker start
root@magpie:~# service docker status
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled)
   Active: active (running) since Sat 2016-06-04 12:24:45 EDT; 1min 33s ago
     Docs: https://docs.docker.com
 Main PID: 6134 (docker)
   CGroup: /system.slice/docker.service
           ├─6134 /usr/bin/docker daemon -H fd://
           └─6140 docker-containerd -l /var/run/docker/libcontainerd/docker-containerd.sock --runtime docker-runc --start-timeout 2m

Jun 04 12:24:45 magpie docker[6134]: time="2016-06-04T12:24:45.089252826-04:00" level=warning msg="mountpoint for pids not found"
Jun 04 12:24:45 magpie docker[6134]: time="2016-06-04T12:24:45.089762916-04:00" level=info msg="Loading containers: start."
Jun 04 12:24:45 magpie docker[6134]: time="2016-06-04T12:24:45.089878726-04:00" level=info msg="Loading containers: done."
Jun 04 12:24:45 magpie docker[6134]: time="2016-06-04T12:24:45.089924156-04:00" level=info msg="Daemon has completed initialization"
Jun 04 12:24:45 magpie docker[6134]: time="2016-06-04T12:24:45.089962696-04:00" level=info msg="Docker daemon" commit=b9f10c9 graphdriver=aufs version=1.11.2
Jun 04 12:24:45 magpie docker[6134]: time="2016-06-04T12:24:45.115270038-04:00" level=info msg="API listen on /var/run/docker.sock"
Jun 04 12:24:45 magpie systemd[1]: [/lib/systemd/system/docker.service:19] Unknown lvalue 'Delegate' in section 'Service'
Jun 04 12:24:45 magpie systemd[1]: [/lib/systemd/system/docker.service:19] Unknown lvalue 'Delegate' in section 'Service'
Jun 04 12:24:45 magpie systemd[1]: [/lib/systemd/system/docker.service:19] Unknown lvalue 'Delegate' in section 'Service'
Jun 04 12:24:45 magpie systemd[1]: [/lib/systemd/system/docker.service:19] Unknown lvalue 'Delegate' in section 'Service'
root@magpie:~# docker --version
Docker version 1.11.2, build b9f10c9
```

## Off-Site Backups


## Sharing

### `SMB` Share for user data

### `iSCSI` Share for RPi Boot Devices
