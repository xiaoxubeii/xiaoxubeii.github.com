---
layout: post
title: "使用libvirt创建虚拟机并使用bridge"
description: ""
category: articles
tags: [Libvirt]
---
#使用libvirt创建虚拟机并使用bridge
最近在研究Opentack nova和neutron的同时，想深入了解下instace的实际创建流程。因为是使用libvirt作为driver，所以准备先熟悉下libvirt创建虚拟机的流程。
我的内核版本如下：

    $ uname -a
    Linux 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux

我的libvirt版本如下：

    $ virsh --version
    0.10.2
    
首先，先准备一个操作系统镜像用来启动虚拟机，我是直接从http://mirror.catn.com/pub/catn/images/qcow2/centos6.4-x86_64-gold-master.img下载的镜像，账户名和密码为：root，changeme1122。
通过qemu-img可以查看镜像的格式：

    $ qemu-img info centos6.4-x86_64-gold-master.img
    image: centos6.4-x86_64-gold-master.img
    file format: qcow2
    virtual size: 10G (10737418240 bytes)
    disk size: 449M
    cluster_size: 65536
    
其次，我们这里先准备虚拟机的网络。修改eth0的网络设置，并增加br0：

    $ cat /etc/sysconfig/network-scripts/ifcfg-eth0
    DEVICE=eth0
    TYPE=Ethernet
    ONBOOT=yes
    HWADDR=00:21:CC:D2:18:62
    BRIDGE=br0
    
    $ cat /etc/sysconfig/network-scripts/ifcfg-br0
    DEVICE=br0
    TYPE=Bridge//这里注意，B一定要大写
    ONBOOT=yes
    BOOTPROTO=static
    IPADDR=172.16.1.110
    GATEWAY=172.16.1.1
    DNS1=8.8.8.8
    DNS2=8.8.4.4
    
然后重启网络：

    $ service network restart
    Shutting down interface eth0:                              [  OK  ]
    Shutting down loopback interface:                          [  OK  ]
    Bringing up loopback interface:                            [  OK  ]
    Bringing up interface eth0:                                [  OK  ]
    Bringing up interface br0:  Determining if ip address 172.16.1.110 is already in use for device br0...
                                                               [  OK  ]
                                                             
查看网络状态：

    $ ifconfig
    br0       Link encap:Ethernet  HWaddr 00:21:CC:D2:18:62  
              inet addr:172.16.1.110  Bcast:172.16.255.255  Mask:255.255.0.0
              inet6 addr: fe80::221:ccff:fed2:1862/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:105 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0 
              RX bytes:0 (0.0 b)  TX bytes:4626 (4.5 KiB)
    
    eth0      Link encap:Ethernet  HWaddr 00:21:CC:D2:18:62  
              UP BROADCAST MULTICAST  MTU:1500  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000 
              RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)
              Interrupt:20 Memory:f2500000-f2520000 
              
    $ brctl show
    bridge name	bridge id		STP enabled	interfaces
    br0		8000.0021ccd21862	no		    eth0
    
这样，br0就可以连接外网了。

然后为虚拟机创建一个domain的xml，名字为centos.xml：

    <domain type='kvm'>
        <name>centos64</name> 
        <memory>1048576</memory> 
        <currentMemory>1048576</currentMemory> 
        <vcpu>2</vcpu>
        <os>
          <type arch='x86_64' machine='pc'>hvm</type>
          <boot dev='hd'/> 
       </os>
       <features>
         <acpi/>
         <apic/>
         <pae/>
       </features>
       <clock offset='localtime'/>
       <on_poweroff>destroy</on_poweroff>
       <on_reboot>restart</on_reboot>
       <on_crash>destroy</on_crash>
       <devices>
         <emulator>/usr/libexec/qemu-kvm</emulator>
         <disk type='file' device='disk'>
          <driver name='qemu' type='qcow2'/>
           <source file='/var/lib/libvirt/images/centos6.4-x86_64-gold-master.img'/> //镜像位置
           <target bus='virtio' dev='vda'/>
         </disk>
	<interface type="bridge">
	 <mac address="2E:B4:31:61:7F:B7"/> //为虚拟机网卡指定一个mac
	 <source bridge="br0"/> //外网br
	 <target dev="tap0"/> //指定tap设备，如果不指定的话，libvirt会自动创建一个vnet
	</interface>
	 <serial type="pty"/>
        <input type='mouse' bus='ps2'/>
         <graphics type='vnc' port='-1' autoport='yes' listen = '0.0.0.0' keymap='en-us'/>
       </devices>
     </domain>

定义并启动虚拟机：

    $ virsh define centos.xml
    
    $ virsh start centos64 
    Domain centos64 started
    
    $ virsh list
    Id    Name                           State
    ----------------------------------------------------
     1     centos64                       running
     
可以使用vnc来连接虚拟机：

    $ virsh vncdisplay centos64
    :0
最后，在虚拟机中按照外部网络设置网络设备，就可以连接外网了。


