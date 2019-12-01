---
title: "New Lab Infrastructure: Part 2"
date: 2019-11-30T18:58:18-05:00
draft: false
tags: ["domain controller", "home lab", "samba"]
---

This is part two of my effort to configure new lab infrastructure. It was written
mainly for my own reference during the process and later, when I inevitably need
to figure out WTF I did. This part focuses on getting my FreeNAS authenticating
against my DC, and convincing the DC to mount home directories and other shares
from the NAS when clients join.

## Securing Things

Before I get started, I need to make sure that my infrastructure trusts itself.
To get trusted certificates, we have to do some magic with certbot and letsencrypt
to issue certs for the various services.

First, a list of all the things we need certificates for:

