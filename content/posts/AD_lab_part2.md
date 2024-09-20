---
title: "Active Directory Lab with Vagrant - Part 2"
date: 2024-09-20T17:47:23+02:00
draft: true
---

> Install the Domain Controller.

## Introduction

This is __2nd article__ of a 3 part series.

* part 1: [Create a Windows Server 2022 VM with vagrant.](/posts/ad_lab_part1/)
* part 2: [Install the Domain Controller.](https://t.ly/ptO4n)
* part 3: [Add some misconfigurations](https://t.ly/ptO4n), and test them with netexec.


In the [previous article](/posts/ad_lab_part1/) we have created a __Windows Server 2022__ virtual machine using `vagrant`.

Now we are going to **install the _Domain Controller_** with `powershell`.


## Current configuration

If you have followed the previous article, your `Vagrantfile` should look like this:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  # Base box image
  config.vm.box = "StefanScherer/windows_2022"

  # Configure a name for the VM
  config.vm.hostname = "dc01"
  config.vm.define "dc01"

  # Change the name in Virtualbox
  config.vm.provider "virtualbox" do |v|
    v.name = "dc01"
  end

  # Add a static IP on a Host-only network
  IP = "10.0.0.4"
  config.vm.network "private_network", ip: IP
end
```

## Provisioning

In `vagrant`, _provisioning_ is the action of configuring your server.

We have multiple possibilites to do this:

* shell scripts
* ansible
* chef
* etc

### Why _powershell_ and not _ansible_

I had some bad experiencies with _ansible_, some years ago, were I did spend 2 weeks debuging a script to apply a CIS configuration to a 4 Ubuntu servers.  
After the job was done, I remember thinking that it would have take me a lot less time, if had written a bash script directly.

I think _ansible_ is really awesome when you have to configure a lot of servers. And want to share your configuration across multiple servers.  
But for configuring a 1 or 4 servers, I now prefer to stick to a simple scripts.  

It also makes things a bit easier to deploy in our case.

Use what makes you happy 🙂.  
But since a lot of AD labs are made using ansible, I thought it would be interesting to explain why we didn't go that way.

### Okay, actual provisioning please !

Yes, sir !

Let's __create a `scripts`  folder__. We will add our powershell scripts here.

And create a `provision.ps1`  inside.

```bash
mkdir scripts
touch scripts/provision.ps1
```

Your folder should now look like this:

```
ad_lab
├── scripts
│   └── provision.ps1
└── Vagrantfile
```

For now we only have one script, but the idea is to have one file for each configuration we want to apply:

* Installing Active Directory. 
* Adding some users.
* Adding a misconfiguration.

The PowerShell script can be referenced by adding the following line to our `Vagrantfile`.

```ruby
config.vm.provision "shell", path: "scripts/provision.ps1"
```

When reaching this line. Vagrant will execute the referenced script on the VM.

The script will be run during the `vagrant up` command.  
But it can also be run by the command

```bash
vagrant provision
```

It's quite handy to debug and test your PowerShell scripts.

Let's write a dummy script.

``` powershell
Write-Host "Hi mom!"
``

And test it



## Install OpenSSH for debugging (optional)

I know, this wasn't part of the title. But I find it makes things a whole lot easier for writing and testing your scripts.

Replace `provision.ps1` with `install_ssh.ps1`.


