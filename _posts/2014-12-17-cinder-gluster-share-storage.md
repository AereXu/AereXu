---
author: "AereXu"
layout: post
category: Storage
title: Cinder gluster share storage
tagline: by Snail
tags: [glusterfs, cinder]
modified: 2014-12-19
---

This is a basic storage sharing investigation in openstack. The cinder provides block storage to instances and the instances will use this virtual disk as glusterfs backend. Then the instances defined as gluster server will provide shared volumes to other clients. 

<!--more-->

# Introduction 

The diagram is below.  
![Diagram](http://i.imgur.com/ynVavPS.png)  

The OS in instance is based on "Red Hat Enterprise Linux Server release 6.5 (Santiago)".  
The openstack version is icehouse.
The glusterfs version is 3.5.2.  

# Cinder
Cinder is a Block Storage service for OpenStack. It's designed to allow the use of either a reference implementation (LVM) to present storage resources to end users that can be consumed by the OpenStack Compute Project (Nova).  
It separates data storage from compute. The data will able to remain and use after an instance crashed.

## Cinder installation
Cinder is consist of one "service controller" which is tight to openstack controller and one "service node" which provide disk storage. They can be install in one machine if needed.  
The installation process should refer to openstack docs. Here's a link to icehouse version: [http://docs.openstack.org/icehouse/install-guide/install/apt/content/cinder-controller.html](http://docs.openstack.org/icehouse/install-guide/install/apt/content/cinder-controller.html).

## Create cinder block volume
Create two cinder volumes using `cinder create`.   

    # cinder create --display-name cinVol1 1
    +---------------------+--------------------------------------+
    |       Property      |                Value                 |
    +---------------------+--------------------------------------+
    |     attachments     |                  []                  |
    |  availability_zone  |                 nova                 |
    |       bootable      |                false                 |
    |      created_at     |      2014-12-17T09:35:44.017867      |
    | display_description |                 None                 |
    |     display_name    |               cinVol1                |
    |      encrypted      |                False                 |
    |          id         | ec4eea2a-f0d9-4614-bc00-2b1d7fd134f4 |
    |       metadata      |                  {}                  |
    |         size        |                  1                   |
    |     snapshot_id     |                 None                 |
    |     source_volid    |                 None                 |
    |        status       |               creating               |
    |     volume_type     |                 None                 |
    +---------------------+--------------------------------------+
    # cinder create --display-name cinVol2 1
    ...
    
Then two volumes whose size are both 1G are created. Their names are cinVol1 and cinVol2.

## Attach the volume to instance  
Assume the two gluster server instances are named as gluServer1 and gluServer2, the cinVol1 will be attached to gluServer1 at /dev/vdb while cinVol2 to gluServer2 at /dev/vdb.  

    # nova volume-attach gluServer1 ec4eea2a-f0d9-4614-bc00-2b1d7fd134f4 /dev/vdb
    +----------+--------------------------------------+
    | Property | Value                                |
    +----------+--------------------------------------+
    | device   | /dev/vdb                             |
    | id       | ec4eea2a-f0d9-4614-bc00-2b1d7fd134f4 |
    | serverId | e01c6fe5-a77f-4845-85bc-947be466f81b |
    | volumeId | ec4eea2a-f0d9-4614-bc00-2b1d7fd134f4 |
    +----------+--------------------------------------+
    # nova volume-attach gluServer2 2d8f8053-121c-4f07-ab2f-bffa38bab30e /dev/vdb
    ...
    
The ec4eea2a-f0d9-4614-bc00-2b1d7fd134f4 is the VolumeId of cinVol1 and 2d8f8053-121c-4f07-ab2f-bffa38bab30e is cinVol2's.

## Partition and format virtual disk
The volume create from cinder is attached at the /dev/vdb as virtual disk, but has no partition table. Use tool __parted__ to build one.

    # parted -s /dev/vdb mklabel msdos
    # parted -s /dev/vdb mkpart primary 0% 100%
    # parted -s /dev/vdb print
    Model: Virtio Block Device (virtblk)
    Disk /dev/vdb: 1074MB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos

    Number  Start   End     Size    Type     File system  Flags
    1      1049kB  1074MB  1073MB  primary
    # mkfs.ext4 /dev/vdb1
    ...
    
Then the partition is at /dev/vdb1 and the file system is ext4.

## Mount disk
To use this disk, it must to be mounted first. Assume glusterfs will use /vdisk as its storage path later.

    # mkdir /vdisk
    # mount -t ext4 /dev/vdb1 /vdisk/
    # df -h | grep vdb1
    /dev/vdb1           1007M   18M  939M   2% /vdisk
    
If want the system to auto mount it at starting up, add an entry of `/dev/vdb1 /vdisk ext4 defaults 0 0` to /etc/fstab.  

# Installation and configuration in glusterfs

## Concepts in glusterfs

### Terms
 1. brick  
The brick is the storage filesystem that has been assigned to a volume.  
 2. client  
The machine which mounts the volume (this may also be a server).  
 3. server  
The machine (virtual or bare metal) which hosts the actual filesystem in which data will be stored.  
 4. subvolume  
a brick after being processed by at least one translator.  
 5. volume  
The final share after it passes through all the translators.  

### Translator  
A translator connects to one or more subvolumes, does something with them, and offers a subvolume connection.  
![Translator](http://i.imgur.com/lr0zXCy.png)  

### Distribute  
Distribute takes a list of subvolumes and distributes files across them, effectively making one single larger storage volume from a series of smaller ones.  

### Replicate  
Replicate is used for providing redundancy to both storage and generally to availability.  

### Stripes  
Stripes data across bricks in the volume. For best results, you should use striped volumes only in high concurrency environments accessing very large files.  

## Installation of glusterfs  
It is recommend to use yum install. Add below lines into a yum repository file.  

    [rhel65_gluster]
    name=RHEL 6.5 gluster
    baseurl=http://download.gluster.org/pub/gluster/glusterfs/3.5/3.5.2/RHEL/epel-6.5/$basearch/
    enabled=1
    gpgcheck=0
    proxy=http://proxy.example.com:8080   #Add if needed

Then execute `yum make cache` to update yum repository.  

### glusterfs server  
To install a glusterfs server, execute `yum install glusterfs{-server,-fuse} -y` and wait the installation to complete.  
Examine the installation using command `gluster --version`. The output should be like:  
 glusterfs 3.5.2 built on Jul 31 2014 18:47:52  
 Repository revision: git://git.gluster.com/glusterfs.git  
 Copyright (c) 2006-2011 Gluster Inc. <http://www.gluster.com>  
 ...  

### glusterfs client
To install a glusterfs client, execute `yum install glusterfs{,-fuse} -y` and wait the installation to complete.  
Examine the installation using command `glusterfs --version`. The output should be like:  
 glusterfs 3.5.2 built on Jul 31 2014 18:47:52  
 Repository revision: git://git.gluster.com/glusterfs.git  
 Copyright (c) 2006-2013 Red Hat, Inc. <http://www.redhat.com/>  
 ...  

## Configuration of glusterfs  

### glusterfs server  

#### start Daemon of gluster
After installation, the glusterfs service keeps stopped. Execute `service glusterd start` to start the daemon and configure it to automatically start by using `chkconfig glusterd on`.  

#### Configure iptables
The gluster server will use certain ports as follows:  
- 24007 TCP for the Gluster Daemon  
- 24008 TCP for Infiniband management (optional unless you are using IB)  
- One TCP port for each brick in a volume. So, for example, if you have 4 bricks in a volume, port 24009 – 24012 would be used in GlusterFS 3.3 & below, 49152 - 49155 from GlusterFS 3.4 & later.  
- 38465, 38466 and 38467 TCP for the inline Gluster NFS server.  
- Additionally, port 111 TCP and UDP (since always) and port 2049 TCP-only (from GlusterFS 3.4 & later) are used for port mapper and should be open.  
The gluster version here is 3.5.2 thus the rules of iptables could be added as below.  

    # iptables -A INPUT -m state --state NEW -m udp -p udp --dport 111 -j ACCEPT
    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 111 -j ACCEPT
    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 666 -j ACCEPT
    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 2049 -j ACCEPT
    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 24007:24008 -j ACCEPT
    # iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 49152:49155 -j ACCEPT

Then execute `service iptables save` and `service iptables restart`.  
Try `service iptables status` to check if the rules is in use.  

#### Creating Trusted Storage Pools
The glusterfs can't working alone. It needs at least two server connecting as a storage pool.  
Here assumes that there are two servers which one server is called "gluServer1" who's IP is "192.168.10.217"  while the other one is called "gluServer2" its IP is "192.168.10.218".  
Add two lines below into /etc/hosts to let the servers know each other.  

    192.168.10.217 gluServer1
    192.168.10.218 gluServer2

Then execute `gluster peer probe gluServer2` in gluServer1. The response should be like __peer probe: success.__.  
Verify the peer status from the gluServer1 using the following commands:  

    # gluster peer status
    Number of Peers: 1
    
    Hostname: gluServer2
    Uuid: 1f318326-7636-4ef6-94c0-0a6e95a816af
    State: Peer in Cluster (Connected)

This can be did at gluServer2 too and the only difference should be the Hostname and Uuid.  
Here's the way to check the storage pool.

    # gluster pool list
    UUID                                    Hostname        State
    1f318326-7636-4ef6-94c0-0a6e95a816af    gluServer1      Connected
    e621d43b-c5ee-4ac9-ae75-047b8b4bd3fd    localhost       Connected

#### Create and configure a glusterfs volume
Here assumes that the /vdisk is for storage and mounted as some kind of file system (xfs, ext3, ...) from other hard disk or virtual disk.  
Make sure the user to command glusterfs is able to read and write in /vdisk.  
Here are 5 types of volumes can be created, [Distributed](http://gluster.org/community/documentation/index.php/Gluster_3.2:_Creating_Distributed_Volumes), [Replicated](http://gluster.org/community/documentation/index.php/Gluster_3.2:_Creating_Replicated_Volumes), [Striped](http://gluster.org/community/documentation/index.php/Gluster_3.2:_Creating_Striped_Volumes), [Distributed Striped](http://gluster.org/community/documentation/index.php/Gluster_3.2:_Creating_Distributed_Striped_Volumes), [Distributed Replicated](http://gluster.org/community/documentation/index.php/Gluster_3.2:_Creating_Distributed_Replicated_Volumes).  
A simple example of Replicated type is given.  

    # gluster volume create gluV1 replica 2 gluServer1:/vdisk/brick1 gluServer2:/vdisk/brick1
    volume create: gluV1: success: please start the volume to access data
    # gluster volume start gluV1
    volume start: gluV1: success
    # gluster volume info gluV1
    Volume Name: gluV1
    Type: Replicate
    Volume ID: af7b78df-e13f-4780-aa87-cc1ebcf2b009
    Status: Started
    Number of Bricks: 1 x 2 = 2
    Transport-type: tcp
    Bricks:
    Brick1: gluServer1:/vdisk/brick1
    Brick2: gluServer2:/vdisk/brick1
    # gluster volume set gluV1 network.ping-timeout 2
    volume set: success
    # gluster volume set gluV1 performance.cache-size 512MB
    volume set: success
    
Now the volume called __gluV1__ is ready. Use a client to access this volume.  
The `gluster volume set` will do some special options to a gluster volume. More detailed information is [here](http://gluster.org/community/documentation/index.php/Gluster_3.2:_Setting_Volume_Options).

### glusterfs client

#### Configure iptables
Add rules of iptables as the server.

#### Enable fuse
After installation the fuse module is not loaded by kernal. Let the fuse be loaded.  

    # modprobe fuse
    # dmesg | grep fuse
    fuse init (API version 7.13)

#### Configure hosts  
Add two lines below into /etc/hosts to let the client know all the gluster servers.  

    192.168.10.217 gluServer1
    192.168.10.218 gluServer2

#### Mount the gluster volume
Assumes the shared folder is called /dataShare and the volume will be mounted there.  

    # mount -t glusterfs -o defaults gluServer1:/gluV1 /dataShare
    # df -h | grep gluV1
    gluServer1:/gluV1 1008M   18M  939M   2% /dataShare
    
The -o means additional options when mounting a glusterfs volume. More detailed information is [here](https://access.redhat.com/documentation/en-US/Red_Hat_Storage/2.0/html/Administration_Guide/chap-Administration_Guide-GlusterFS_Client.html#sect-Administration_Guide-GlusterFS_Client-GlusterFS_Client-Mounting_Volumes).

# Basic file tests

## Clients file tests
Make sure that the users share the same UID on both glustet clients or one client will not able to edit a file created by another client. Or, use __POSIX Access Control Lists (ACLs)__ to manage the permissions for different users or groups. [Doc](http://gluster.org/community/documentation/index.php/Gluster_3.2:_POSIX_Access_Control_Lists)  
At client1, do `echo "Hello! This is client1 at $(date)" >> /dataShare/hello.log` and switch to client2 to execute `echo "Hello! This is client2 at $(date)" >> /dataShare/hello.log`.  
Then back to client1 and check the hello.log file.  

    # cat /dataShare/hello.log
    Hello! This is client1 at Wed Dec 17 22:22:29 EST 2014
    Hello! This is client2 at Wed Dec 17 22:22:33 EST 2014

## Server file tests
As we know, the replicated volume "gluV1" of glusterfs is placed at /vdisk/brick1. After the clients file tests, the file "hello.log" are stored on both servers.  
It's not allowed to edit these files directly. Executing `echo "Hello! This is server1 at $(date)" >> /vdisk/brick1/hello.log` will ruin the "hello.log" file to all clients.

    # cat hello.log
    cat: hello.log: Input/output error

# Manila, a better solution?
Manila is a openstack service that provides coordinated access to shared or distributed file systems. It will be graduated and submit for consideration as a core service as early as the Kilo release.  
It is original based on cinder and also consist of a controller and a storage node.  
Manila will create a Neutron private network and subnet or using the existing one as the share network. It allows to add access according to ips. If one instance is allowed to access the shared volume, it can mount the volume using  
`mount -t nfs 10.254.0.3:/shares/share-4ed16b64-59ee-4123-b6f0-f42148280a99 /mnt/sharedVolume`.  
The __10.254.0.3:/shares/share-4ed16b64-59ee-4123-b6f0-f42148280a99__ is the export location which will be provide by malina service.  
    
    # manila list
    +--------------------------------------+----------------+------+-------------+-----------+---------------------------------------------------------------+
    |                  ID                  |      Name      | Size | Share Proto |   Status  |                        Export location                        |
    +--------------------------------------+----------------+------+-------------+-----------+---------------------------------------------------------------+
    | 4ed16b64-59ee-4123-b6f0-f42148280a99 |    myshare     |  1   |     NFS     | available | 10.254.0.3:/shares/share-4ed16b64-59ee-4123-b6f0-f42148280a99 |
    | e9b92837-ea2b-4acd-ab65-65c98199e78b | devstack_share |  1   |     NFS     | available | 10.254.0.5:/shares/share-e9b92837-ea2b-4acd-ab65-65c98199e78b |
    +--------------------------------------+----------------+------+-------------+-----------+---------------------------------------------------------------+

With the service of manila, the diagram of shared storage between instances can be simplify as below.
![manila](http://i.imgur.com/rgVFkh1.png)

Monila supports NFS now and release a demo based on Openstack icehouse version using Devstack. Here are the two demo links.  
 1. [https://wiki.openstack.org/wiki/Manila/IcehouseDevstack](https://wiki.openstack.org/wiki/Manila/IcehouseDevstack)  
 2. [http://netapp.github.io/openstack/2014/08/15/manila-devstack/](http://netapp.github.io/openstack/2014/08/15/manila-devstack/)

# Other informations

## Swift, openstack object storage 
Swift is a core service who provides highly available, distributed, eventually consistent object/blob storage. It uses RESTful API to transfer data. It gets great advantages in cloud storage environment for it simplifies data as objects in containers. [For more](http://docs.openstack.org/developer/swift/).

## Fabrix Scale Out NAS
Fabrix is pioneering a revolutionary, cloud based scale out storage and computing platform focused on providing a simple, tightly integrated solution optimized for media storage, processing and delivery applications such as Cloud DVR, and VOD expansion.  
Ericsson has acquired Fabrix in the goal of accelerating cloud video transformation.  
Fabrix's storage technology gets many advantages like unified clustered storage, distributed computing and shared nothing architecture. [For more](http://www.fabrixsystems.com/technology/scale-out-storage).


# Reference
 1. __Volume type of gluster introdution__: http://gluster.org/community/documentation/index.php/Gluster_3.2:_Setting_Up_GlusterFS_Server_Volumes  
 2. __Options of `gluster volume set`__: http://gluster.org/community/documentation/index.php/Gluster_3.2:_Setting_Volume_Options  
 3. __Options of `mount -t glusterfs -o`__: https://access.redhat.com/documentation/en-US/Red_Hat_Storage/2.0/html/Administration_Guide/chap-Administration_Guide-GlusterFS_Client.html#sect-Administration_Guide-GlusterFS_Client-GlusterFS_Client-Mounting_Volumes  
