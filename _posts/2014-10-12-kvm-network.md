---
author: "AereXu"
layout: post
category: KVM
title: KVM network refer doc
tagline: by Snail
tags: [KVM, libvirt]
modified: 2014-08-27
---

Introduction of configure virtual network and KVM based on libvirt.

<!--more-->

# Introduction

This is a reference document of building KVMs with certain networks to support BMC deployment virtualization.

It requires the basic knowledge of Linux bridge, net-configure interface, `qemu-kvm` and `libvirt`.  

# Network physical diagram
![](http://i.imgur.com/qlUcIUd.png) 

# KVM Install 
* OS: This document is based on Red Hat Enterprise Linux 6.5
* Hardware: 
 * CPU supports __Virtualization Technology__ 
 * Memory more than __4G__
 * Harddisk more than __10G__
* User: The user to do below steps should be __root__ or gets __sudo__ authority. 

## Install dependencies and tools
To make a check your machine supports KVM, use grep as below  

    grep -E '(vmx|svm)' /proc/cpuinfo
    flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc rep_good unfair_spinlock pni pclmulqdq vmx ssse3 cx16 pcid sse4_1 sse4_2 x2apic popcnt aes hypervisor lahf_lm vnmi ept
    <!-- more information is ignored -->
    
If the output is not empty, use yum to install dependencies and tools as below  

    yum install qemu-kvm qemu-kvm-tools libguestfs-tools -y
    yum groupinstall Virtualization-Platform Virtualization-Clients Virtualization-Tools -y

To verify the installation, execute  

    /usr/libexec/qemu-kvm --version
    
The output message should be like "__QEMU PC emulator version 0.12.1 (qemu-kvm-0.12.1.2), Copyright (c) 2003-2008 Fabrice Bellard__".


# Host network configuration 
* NetworkManager: The NetworkManager service must be stopped or it'll interferes the condition of networks. Execute `service NetworkManager status` and if get message as "NetworkManager (pid $pid_numb) is running..." then execute `service NetworkManager stop` 
* Iptables: If your new network can't access the Internet, check the rules of your iptables or shut down your iptables by executing `service iptables stop`

## The Configuration of no bonding
### Setting up a linux bridge 
Go into network configuration directory.  
  `cd /etc/sysconfig/network-scripts`  
Create a configure file named as ifcfg-br\*, \* must be a number. Here use ifcfg-br1 as an example.  
Example 1: 

    #This is used when there's a DHCP server.
    DEVICE=br1
    ONBOOT=yes
    BOOTPROTO=dhcp
    TYPE=Bridge
    STP=on

Example 2:

    #This is used when there's no DHCP server.
    DEVICE=br1
    ONBOOT=yes
    BOOTPROTO=static
    IPADDR=192.168.100.16
    NETMASK=255.255.255.0
    TYPE=Bridge
    STP=on

### Configurate the eth
Here assume that eth1 is linked to the br1 above.  
Create a configure file named as ifcfg-eth1.  

    DEVICE=eth1
    TYPE=Ethernet
    BRIDGE=br1
    ONBOOT=yes
    BOOTPROTO=none
    
### Validate the linux bridge
To now, the linux bridge is completely setted but not in validation. To validate it, execute `service network restart`.  
Ensure that the __NetworkManager__ is closed before doing this.  
If any errors occurs, do `rm /etc/udev/rules.d/70-persistent-net.rules` and try again.  
If no errors occur, execute `brctl show` to get the information as follow:  

    bridge name    bridge id            STP enabled     interfaces  
    br1            8000.525400117fda    yes             eth1

### Using libvirt tools to build virtual networks
#### Definition of virtual network
Libvirt use xml format file to define a network. [For more](http://libvirt.org/formatnetwork.html#elementsAddress)  
This example showes a bridge forward network. The libvirt will create a __tap__ device and bond it to a bridge. Then the guest virtual machine will use this tap as its ethernet device.    

    <network>
        <name>virnet-1</name>
        <forward mode="bridge"/>
        <bridge name="br1"/>
    </network>
    
* __name__: The network name for libvirt. You can use `virsh net-list --all` to list all and `virsh net-info $netname` to get more information.
* __bridge__: The name of the to be bonded bridge. After the net is started by libvirt, you can check it: `ifconfig $netname`.
* __forward__: The forward mode of the virtual network. To use linux bridge, the mode should be "bridge".
* __bridge__: When forward mode is setted as bridge, use this to assign the linux bridge to virtual network. The name is the device name in ifcfg-br?.

#### Active the virtual network
`virsh net-create $xml_file` then a temporaty virtual network is created. If you this net to be automatically started at boot, then `virsh net-autostart $netname`.
## The Configuration including bonds
Most of the steps are the same with no bonding one. The difference is as follow steps.
### Setting up a linux bridge 
Please refer to the part of setting up a lunix bridge in "no bonding". They are exactly the same as those steps. 
### Setting up a bonding
Go into network configuration directory.  
`cd /etc/sysconfig/network-scripts`  
Create a configure file named as ifcfg-bond\*, \* must be a number. Here use ifcfg-bond2 as an example.

    DEVICE=bond2
    TPYE=Bonding
    BRIDGE=br2
    BONDING_OPTS="mode=1 miimon=200"
    ONBOOT=yes
    BOOTPROTO=none
    USERCTL=no

* Above assumes that this bond2 will link to a linux bridge named br2.
* The BONDING_OPTS give the detailed options of bonding. Here the "mode 1" means that one is active while others ara backup and the "miimon=200" means the MII will verify if a NIC is active every 200 milliseconds. To know more, go to [Redhat site](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/sec-Specific_Kernel_Module_Capabilities.html#sec-Using_Channel_Bonding).
* The USERCTL=no means user except root can not configure this net device.  

### Configurate the eth
Here assume that eth2 and eth3 is bonded to the bond2 above.  
Create a configure file named as ifcfg-eth2 first.

    DEVICE=eth2
    TYPE=Ethernet
    ONBOOT=yes
    BOOTPROTO=none
    MASTER=bond2
    SLAVE=yes

Then a ifcfg-eth3.

    DEVICE=eth3
    TYPE=Ethernet
    ONBOOT=yes
    BOOTPROTO=none
    MASTER=bond2
    SLAVE=yes

### Others
The rest steps like manage linux bridges and virtual networks are the same as "The Configuration of no bonding".
# Create a KVM 
## Create a KVM using an existing image
Assume that there's an image named "bmc15b-Fri.qcow2" which is created from some other KVM's snapshot.

    virt-install \
        -n BMC-release \
        -r 512 \
        --vcpus 1 \
        --network network=virnet-1 \
        --graphics vnc \
        --hvm \
        --virt-type kvm \
        --os-type linux --os-variant rhel6 \
        --disk path=/var/tmp/bmc15b-Fri.qcow2,format=qcow2 \
        --cdrom /var/tmp/seed.iso \
        --boot hd

Then a virtual machine named BMC-release which get 512MB ram, one virtual CPU and virnet-1 as its eth0 is created.  

* __--network network=vnet-1__: When you create a virtual machine, this option means VM will use a tap device named vnet-1 as its default net device link to the __br1__. 
* __--cdrom /var/tmp/seed.iso__: File or device use as a virtual CD-ROM device for fully virtualized guests.
* __--boot hd__: A __boot__ defines where the VM will boot from. Here it boots from hd(hard disk).

## Auto install a OS base image as raw format
Assume that there's a official rhel iso named rhel-server-6.5-x86_64-dvd.iso, you can use a premade ks.cfg file to do auto-install.  

    virt-install \
        -n BMC-release \
        -r 1024 \
        --vcpus 1 \
        --network network=virnet-1 \
        --graphics vnc \
        --hvm \
        --virt-type kvm \
        --os-type linux --os-variant rhel6 \
        --disk path=/var/tmp/bmc15b-Fri.img,size=20G 
        --location ${ISO_PATH}/rhel-server-6.5-x86_64-dvd.iso  \
        --initrd-inject=ks.cfg  \
        --extra-args "ks=file:/ks.cfg"

You can view the installation process in vnc viewer.
        
## Edit image using guestfish
Sometimes a image may need to be made some changes before release. The guestfish is a tool to help doing this safely.  
Assume that here gets a RHEL qcow2 image called rhel6.5_demo.img. Mount the image in read-write mode as root, as follows:

    # guestfish --rw -a centos63_desktop.img

    Welcome to guestfish, the libguestfs filesystem interactive shell for
    editing virtual machine filesystems.

    Type: 'help' for help on commands
        'man' to read the manual
        'quit' to quit the shell

    ><fs>
    
Then use `run` command to launch a lightweight virtual machine.  

    ><fs> run

If viewing the file systems in the image is requited, using the `list-filesystems` command.  
   
    ><fs> list-filesystems
    /dev/vda1: ext4
 
Mount the disk or volumn and start to edit files.  

    ><fs> mount /dev/vda1 /
    ><fs> rm /etc/udev/rules.d/70-persistent-net.rules
    ><fs> edit /etc/sysconfig/network-scripts/ifcfg-eth0

When everything is done, use `exit` to leave.  
    ><fs> exit    
    
# Reference
 1. [Libvirt virtual network]: http://libvirt.org/formatnetwork.html#elementsAddress
 2. [Bonding options]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/sec-Specific_Kernel_Module_Capabilities.html#sec-Using_Channel_Bonding).
 3. [Doc of virt-install]: http://linux.die.net/man/1/virt-install
 4. [Modify images]: http://docs.openstack.org/image-guide/content/ch_modifying_images.html