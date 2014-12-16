---
author: "AereXu"
layout: post
category: openvswitch
title: OpenvSwitch refer doc
tagline: by Snail
tags: [openvswitch, sFlow]
modified: 2014-10-27
---

Install openvswitch and configure sFlow

<!--more-->

# Installation  

## Install openvswitch for rhel-6.5

    yum install wget openssl-devel
    yum groupinstall "Development Tools"

    adduser ovswitch
    su ovswitch

    cd ~
    wget http://openvswitch.org/releases/openvswitch-2.1.3.tar.gz
    tar -zxvf openvswitch-2.1.3.tar.gz
    cd openvswitch-2.1.3
    mkdir -p /home/ovswitch/rpmbuild/SOURCES
    cp ../openvswitch-2.1.3.tar.gz /home/ovswitch/rpmbuild/SOURCES/
    cp rhel/openvswitch-kmod.files /home/ovswitch/rpmbuild/SOURCES/
    rpmbuild -bb rhel/openvswitch.spec
    rpmbuild -bb rhel/openvswitch-kmod-rhel6.spec
    exit

    yum localinstall /home/ovswitch/rpmbuild/RPMS/x86_64/kmod-openvswitch-2.1.3-1.el6.x86_64.rpm -y
    yum localinstall /home/ovswitch/rpmbuild/RPMS/x86_64/openvswitch-2.1.3-1.x86_64.rpm -y

After the installation is done, execute `modinfo openvswitch` and `modinfo bridge`. Compare the "vermagic" lines output to ensure they are exactly the same.
## Configurate openvswitch
    
    modprobe openvswitch
    mkdir -p /etc/openvswitch
    ovsdb-tool create /etc/openvswitch/conf.db /usr/share/openvswitch/vswitch.ovsschema
    mkdir -p /var/run/openvswitch/ 
    ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock \
                 --remote=db:Open_vSwitch,Open_vSwitch,manager_options \
                 --private-key=db:Open_vSwitch,SSL,private_key \
                 --certificate=db:Open_vSwitch,SSL,certificate \
                 --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert \
                 --pidfile --detach
    service openvswitch restart
    ovs-vsctl --no-wait init

# Using openvswitch in KVM    
## Make a ovs bridge
Basically the ovs bridge is just like the linux bridge but is more powerful. For instance, it supports VLan tag.  
To create a ovs-bridge, execute `ovs-vsctl add-br $BRIDGE`. To link some Ethernet device to it, execute `ovs-vsctl add-port $BRIDGE $ETH-DEV`. 
To check the status of the ovs-bridge, execute `ovs-vsctl add-br show` 
## Monitoring VM traffic using sFlow
### Example diagram
![Alt teasdfxt](http://openvswitch.org/support/config-cookbooks/sflow/sflow-setup.png)
### Build ovs-bridge and VMs
Build a ovs-bridge named br0 and link it to eth0.  

    ovs-vsctl add-br br0
    ovs-vsctl add-port br0 eth0
    
Use `ovs-vsctl show` to check the status.

### Configuration file
Touch a file names as __sflow-conf.sh__ and `chmod +x sflow-conf.sh`. Then add beloe contents into it.

    #!/bin/bash
    COLLECTOR_IP=10.120.0.183
    COLLECTOR_PORT=6343
    AGENT_IP=eth1
    HEADER_BYTES=128
    SAMPLING_N=64
    POLLING_SECS=10

* Port 6343 (COLLECTOR_PORT) is the default port number for sFlowTrend.  
* IP 10.120.0.183 (COLLECTOR_IP) the IP address for the collector of the AGENT_IP. 
* Device name eth1 (AGENT_IP) is a value indicates that the sFlow agent should send traffic from eth1's IP address. 
* The other values indicate settings regarding the frequency and type of packet sampling that sFlow should perform.    

### Add rule to the ovs-bridge
On Host1, run the following command to create an sFlow configuration and attach it to bridge br0.

    ovs-vsctl -- --id=@sflow create sflow agent=${AGENT_IP} target=\"${COLLECTOR_IP}:${COLLECTOR_PORT}\" header=${HEADER_BYTES} sampling=${SAMPLING_N} polling=${POLLING_SECS} -- set bridge br0 sflow=@sflow
To remove sFlow configuration from a bridge (in this case, "br0"), run this command, where "sFlow UUID" is the UUID returned by the command used to set the sFlow configuration initially:

    ovs-vsctl remove bridge br0 sflow <sFlow UUID>

To see all current sets of sFlow configuration parameters, run:

    ovs-vsctl list sflow
    