---
title: "Fix_virtualbox"
date: 2024-09-25T11:52:27+02:00
draft: true
---

## The bug

One infamous bug often encountered on Linux when using Virtualbox is the error :``

*Add a screenshot*

Virtualbox will tell you to run the command

```bash
sudo /sbin/vboxconfig
```

But you will likely be meet with another error.

## Root cause

Virtualbox relies on a bunch of kernel modules to handle virtualization:

```bash
❯ lsmod | grep vbox
vboxnetadp             28672  0
vboxnetflt             32768  1
vboxdrv               696320  4 vboxnetadp,vboxnetflt
```

When there is a kernel upgrade, for some reason the kernel module aren't created automaticaly.
And so Virtualbox can't run without them, hence the error.

## Fix

You can reinstall the required kernel modules with the following command:

```bash
sudo apt reinstall linux-headers-generic linux-headers-$(uname -r)
```

And once this is done, you just have to reload the virtualbox modules with

```bash
sudo /sbin/vboxconfig
```

And everything should be working again.

## Things I am not so sure about

Nowadays, I use the newest version of VirtualBox directly from [VirtualBox repositories](https://www.virtualbox.org/wiki/Linux_Downloads#Debian-basedLinuxdistributions).

But when I was using it from the distribution repositories. I was also reinstalling the `dkms` package.

```bash
sudo apt install --reinstall linux-headers-$(uname -r) virtualbox-dkms dkms
```

Not sure if this is required.

