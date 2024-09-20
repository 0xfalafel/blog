---
title: "Create an Active Directory Lab with Vagrant - Part 1"
date: 2024-09-20T10:09:12+02:00
draft: true
---

# Create a Windows Server 2022 VM with vagrant

## Introduction

We are going to see how you can easily create an __Active Directory__ home __lab__ with __Vagrant__, __VirtualBox__ and __Powershell__.  


This is a 3 part series. In __this article__, will use __Vagrant__ to create an __Windows Server 2022__.

* part 1: [Create a Windows Server 2022 VM with vagrant.]()
* part 2: [Install the Domain Controller.]()
* part 3: [Add some misconfigurations](), and test them with netexec.


If you are __already familiar with Vagrant__, you might want to __skip__ directly __to__ the [install the Domain Controler]() section.

## Installation

First, you need to __install [vagrant](https://developer.hashicorp.com/vagrant/install)__ and __[virtualbox](https://www.virtualbox.org/wiki/Downloads)__.


If you are on Linux, you can probably use the __packages from your distribution__, as we don't rely on any new features. Homebrew should also do the job.

My setup uses `Vagrant 2.4.1`, and `VirtualBox 7.0.18` on a `Ubuntu 22.04` laptop. But again, we don't rely on any specific feature, so whatever should do.

You can also use _VMware_ if you want, but in this tutorial we will use _VirtualBox_.  
We won't use _ansible_ in this tutorial to keep things easy to deploy.  

## Create a Windows Server

This part is surprisingly easy.

Let's create a __folder `ad_lab`__ for our project, and create a __`Vagrantfile`__ inside it.  
Your folder should look like this:

```
ad_lab
└── Vagrantfile
```

Put the following content in your `Vagrantfile`:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  # Base box image
  config.vm.box = "StefanScherer/windows_2022"
end
```

Run the command __`vagrant up`__ inside the folder, and _voilà_ you've created a ___Windows 2022 server___ Virtual machine.

![The VM created with `vagrant up`](/ad_lab/vagrant_up_firstrun.png)

#### What did we just do ?

`Vagrant` lets you easily create Virtual Machine on your _local computer_, on a _proxmox_ or a cloud provider.

The main idea is to use ___infrastructure as code___ and to use scripts to create reproducible environments.

Stefan Scherer has created a brunch of [Windows VM template](https://app.vagrantup.com/StefanScherer) from the evaluation versions of Windows.  
Since things aren't going to run for months, and we can recreate the lab with `vagrant up` so we don't mind using an evaluation version of Windows.

### Clean things up

You can __delete the VM__ you've created by running the following command from the `ad_lab` folder:

```bash
vagrant destroy
```


## Customize our VM

Let's customize our VM a bit by giving it a __name__, and an __IP address__.

### Name

Add the following to your `Vagrantfile`:

```ruby
# Configure a name for the VM
config.vm.hostname = "dc01"
config.vm.define "dc01"
```

* With __`vm.hostname`__, the VM will use `dc01` has her __hostname__.
* __`config.vm.define`__ is used to give __a name__ to the virtual machine. So when we start our VM with `vagrant up`, the name is `dc01` instead of `default`:

```bash
❯ vagrant up
Bringing machine 'dc01' up with 'virtualbox' provider...
==> dc01: Importing base box 'StefanScherer/windows_2022'...
```

But the name in VirtualBox is still a bit weird, like `ad_lab_dc01_1726832429414_18289`.

If we want a specific __name in VirtualBox__, we can use some instructions specific to this _provider_.

```ruby
# Setup the name in virtualbox
config.vm.provider "virtualbox" do |v|
  v.name = "dc01"
end
```

The ___provider___ is the __tool we use to build our VM__. Vagrant also supports VMware, docker, proxmox, etc… VirtualBox is the default _provider_.


`config.vm.provider "virtualbox"` defines some instructions for VirtualBox.


### IP address

Setting up an IP address is pretty straightforward.  
Add this to the `Vagrantfile`:

```ruby
# Add a static IP on a host-only network
IP = "192.168.56.4"
config.vm.network "private_network", ip: IP
```

Vagrant will be unhappy if the `IP` is not in the IP range of your _virtual network manager_. Just take an IP that matches.

![Erroneous IP address](/ad_lab/vagrant_ip_error.png)

### Result

![Vagrant with a configured Windows 2022 server](/ad_lab/vagrant_up_named2.png)

By now your __`Vagrantfile`__ should look like this:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  # Base box image
  config.vm.box = "StefanScherer/windows_2022"

  # Configure a name for the VM
  config.vm.hostname = "dc01"
  config.vm.define "dc01"

  # Change the name in VirtualBox
  config.vm.provider "virtualbox" do |v|
    v.name = "dc01"
  end

  # Add a static IP on a host-only network
  IP = "192.168.56.4"
  config.vm.network "private_network", ip: IP
end
```

## Conclusion

In this tutorial, we have created a __Virtual Machine__ of __Windows 2022 Server__ using `vagrant`.

In the __[next part]()__, we will _provision_ the VM. And __install Active Directory__ using __Powershell__.