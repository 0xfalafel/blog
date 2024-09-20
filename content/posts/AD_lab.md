---
title: "Active Directory Lab with Vagrant"
date: 2024-09-20T10:09:12+02:00
draft: true
---

## Introduction

In this blog post, we are going to see how you can easily create an __Active Directory__ home __lab__ with __Vagrant__, __Virtualbox__ and __Powershell__.

You can also use _VMware_ if you want, but we will use _Virtualbox_ in this tutorial.  
We won't use _ansible_ in this tutorial to keep things easy to deploy.  

## Installation

First you need to __install [vagrant](https://developer.hashicorp.com/vagrant/install)__ and __[virtualbox](https://www.virtualbox.org/wiki/Downloads)__.


If you are on linux, you can probably use the __packages from your distribution__, as we don't rely on any new features. Homebrew should also do the job.

My setup use `Vagrant 2.4.1`, and `Virtualbox 7.0.18` on a `Ubuntu 22.04` laptop. But again, we don't rely on any specific feature, so whatever should do.


## Create a Windows Server

This part is suprisingly easy.

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

`Vagrant` let's you easily create Virtual machine on you _local computer_, on a _proxmox_ or a cloud provider.

The main idea is to use ___infrastructure as code___ and to use scripts to create reproductible environments.

Stefan Scherer has created a brunch of [Windows VM template](https://app.vagrantup.com/StefanScherer) from the evalution versions of Windows.  
Things aren't going to run for months, and we can recreate the lab with `vagrant up` so we don't mind using an evaluation version of Windows.

### Clean things up

You can __delete the VM__ you've created by running the following commnand from the `ad_lab` folder:

```bash
vagrant destroy
```


## Customize our VM

Let's customize our VM a bit by giving it a __name__, and an __IP address__.



