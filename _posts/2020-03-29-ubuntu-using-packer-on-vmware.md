---
title: "Deploy Ubuntu using Packer on VMWare ESXi"
layout: post
excerpt: "Let's deploy a Ubuntu VM from ISO image on VMWare ESXi using Packer "
tags:
- packer
- vmware
- vsphere
- esxi
- vcenter
---

## Requirements

* An ESXi host attached to vCenter. I am using ESXi v6.7
* Clone  <https://github.com/amard33p/packer101/>

## Builder - vsphere-iso or vmware-iso?

Packer supports 2 builders to create VM from an ISO image on VMWare.
- [vsphere-iso](https://packer.io/docs/builders/vsphere-iso.html): ESXi host must be attached to a vCenter. Uses vSphere API. 
- [vmware-iso](https://packer.io/docs/builders/vmware-iso.html): ESXi host need not be attached to vCenter. However few configurations are needed in ESXi:
  - [Enable GuestIPHack](https://packer.io/docs/builders/vmware-iso.html#:~:text=GuestIPHack)
  - [Enable VNC](http://www.supermaru.com/2017/07/enabling-vnc-esxi/)

This example uses vsphere-iso.  
**Note:** [The latest packer versions support vsphere-iso natively](https://github.com/hashicorp/packer/pull/8480). So the jetbrains plugin is no longer required.

## Usage

1. Setup Packer
  ```
  cd /tmp
  wget https://releases.hashicorp.com/packer/1.5.5/packer_1.5.5_linux_amd64.zip
  unzip packer_1.5.5_linux_amd64.zip
  cp packer /usr/local/bin/
  chmod u+x /usr/local/bin/packer
  ```
2. Edit `ubuntu-18/variables.json`
3. Run `packer build -var-file variables.json ubuntu-18.json`
4. In case [Ubuntu decides to drop floppy driver support](https://github.com/jetbrains-infra/packer-builder-vsphere/issues/176), we can use [packer's inbuilt http-server feature to expose the preseed file](https://github.com/amard33p/packer101/commit/e33cf2103eb74985947b175881fcdbeb20b96b44).

_References:_  
- <https://www.thehumblelab.com/automating-ubuntu-18-packer/>
