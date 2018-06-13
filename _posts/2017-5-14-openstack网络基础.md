---
layout:     post
title:      openstack网络基础
subtitle:
date:       2017-5-14
author:     dodoYuan
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - openstack & 网络基础
---


针对官网关于[网络介绍](https://docs.openstack.org/newton/networking-guide/)
#### 1、基本概念
##### 1） 以太网
```
$ ip link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
     link/ether 08:00:27:b9:88:74 brd ff:ff:ff:ff:ff:ff
```
以太网协议是比较熟悉的第二层协议，给IP数据包加上MAC层头部，每个host通过NIC监听数据是否发送给自己，是则接收，否就丢弃；但是因为一个主机还有很多虚拟机，而虚拟机的MAC地址一般都有自己唯一的，因此需要将网卡进行设置为混杂模式（ promiscuous mode），该模式下网卡也会将数据接收由操作系统进一步处理。
##### 2）VLAN
因为交换机只能分割冲突域，不能分隔广播域，而在一个大型局域网特别是云网络中，广播域都比较大，广播数据会造成网络拥塞，引入VLAN，分隔广播域。

##### 3）ARP
常用命令
```
$ arping -I eth0 10.30.0.132 #查看特定IP地址的MAC地址
$ arp -n #查看主机ARP缓存
```
当不再同一个局域网时，则返回路由器接口MAC地址。
##### 4） DHCP
DHCP是给局域网中主机分配IP地址的功能实体，必须和主机工作在同一个局域网。分为DHCP server和DHCP client，server工作端口号为67，client工作端口为66.以下为主要交互流程：
* 第一步：发送一个探测包，宣告自己的MAC地址，请求分配IP地址，由于并不知道server在哪，因此目的IP为255.255.255.255，即广播地址。
* 第二步：服务器收到后，根据client的MAC地址，回复一个可用的IP地址
* 第三步：回复server，确认IP
* 第四步：server返回确认包

openstack 使用第三方提供的软件 [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html)完成DHCP功能，带有详细的系统日志可以检测请求和回复包。
##### 5） IP
IP是第三层的概念，对应路由器，路由器通过IP寻址，查询FIB路由表。linux上以下命令可以显示路由表：
```
$ ip route show
$ route -n
$ netstat -rn
```
> A [DHCP](https://docs.openstack.org/newton/networking-guide/intro-basic-networking.html#dhcp) server typically transmits the IP address of the default gateway to the DHCP client along with the client’s IP address and a netmask.
这段话如何理解

On a Linux machine, the `traceroute` and more recent `mtr` programs prints out the IP address of each router that an IP packet traverses along its path to its destination.
##### 6) 覆盖网络
overlay技术领域有三大技术路线：VXLAN、NVGRE、STT，其中使用广泛的是VXLAN技术
###### GRE
GRE通用路由封装，三层隧道技术，提出了一种在一种协议中传输另一种协议的方法，即 `X over Y`。对任何一种隧道技术，都分为三个部分，乘客协议，封装协议，传输协议，GRE就是一种封装协议，IP是最常用的运输协议。如IPX报文、 PPP、 MPLS 等协议可以作为乘客协议封装在GRE payload中，交由IP运输协议传输。当然也可以IP中封装IP，一般用来将私有IP在公网上传输，如下：
![image.png](http://upload-images.jianshu.io/upload_images/3635313-ebc7349834102e73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
但又有疑问，问什么那些协议不能单独封装在IP中，而一定要加一个GRE头部呢，就像要将信加个信封，GRE可以起到统一规范和标识的作用，GRE中的头部协议字段进一步区分封装的是什么协议。如下所示，是最新RFC对GRE简化后的GRE结构。
![image.png](http://upload-images.jianshu.io/upload_images/3635313-6f2e5dbe9f323c3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### VxLAN

##### 6）VRF虚拟路由转发
##### 7）网络地址转换
[参考](https://zhuanlan.zhihu.com/p/26992935)
第一个概念，私有IP地址，有三个地址范围，不会在公网上路由。
* 10.0.0.0/8
* 172.16.0.0/12
* 192.168.0.0/16

这些私有地址配合NAT极大缓解了IP地址分配压力，NAT有以下几类：
* SNAT 源网络地址转换，将私网IP地址和端口号映射到公网IP和分配的端口
* DNAT目的网络地址转换
>OpenStack uses DNAT to route packets from instances to the OpenStack metadata service. Applications running inside of instances access the OpenStack metadata service by making HTTP GET requests to a web server with IP address 169.254.169.254. In an OpenStack deployment, there is no host with this IP address. Instead, OpenStack uses DNAT to change the destination IP of these packets so they reach the network interface that a metadata service is listening on.

* one-to-one NAT，一个私网地址对应一个公网IP，openstack用来实现浮动IP

#### 2、openstack 网络
>openstack安装中会主要会遇到两种网络，provider network 和self-service network

###### provider network
Provider networks generally offer simplicity, performance, and reliability at the cost of flexibility. 默认只有administrators可以创建和更新provider network，因为涉及到对物理网络的配置。
Also, provider networks only handle layer-2 connectivity for instances, thus lacking support for features such as routers and floating IP addresses.
###### self-service network
Self-service networks primarily enable general (non-privileged) projects to manage networks without involving administrators.

###### FWaaS
Firewall-as-a-service

##### ML2 plug-in
[reference IBM developer](https://www.ibm.com/developerworks/cn/cloud/library/cl-cn-openstackneutronml2/index.html)
The Modular Layer 2 (ML2) neutron plug-in is a framework allowing OpenStack Networking to simultaneously use the variety of layer 2 networking technologies found in complex real-world data centers. 讲人话，就是ML2是可以同时管理多种layer2技术的框架。
>需要注意的是，ML2 与运行在各个 OpenStack 节点上的 L2 agents 是有区别的。ML2 是 Neutron server 上的模块，而运行在各个 OpenStack 节点上的 L2 agents 是实际与虚拟化 Layer 2 技术交互的服务。ML2 与运行在各个 OpenStack 节点上的 L2 agent 通过 AMQP（Advanced Message Queuing Protocol）进行交互，下发命令并获取信息。

ML2在出现之前有两个主要问题：
* OpenStack Neutron 最多只支持一种 Layer 2 技术，也就是说如果配置使用了 Open vSwitch，那么整个 OpenStack 环境都只能使用 neutron-openvswitch-agent 作为 Layer 2 的管理服务与 Open vSwitch 交互。
* 每支持一种 Layer 2 技术，都需要对 OpenStack Neutron 中的 L2 resource，例如 Network/Subnet/Port 的逻辑进行一次重写，这大大增加了相应的工作量。

>ML2 的提出解决了上面两个问题。ML2 之前的 Layer 2 plugin 代码相同的部分被提取到了 ML2 plugin 中。这样，当一个新的 Layer 2 需要被 Neutron 支持时，只需要实现其特殊部分的代码，需要的代码工作大大减少，开发人员甚至不需要详细了解 Neutron 的具体实现机制，只需要实现对应的接口。并且，ML2 通过其中的 mechanism drivers 可以同时管理多种 Layer 2 技术

##### ML2框架
![image.png](http://upload-images.jianshu.io/upload_images/3635313-38771ef5ba2a5187.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

http://superuser.openstack.org/articles/everything-you-need-to-know-to-get-started-with-neutron-f90e2797-26b7-4d1c-84d8-effef03f11d2/

##### 3、neutron address scope
[参考学习](http://www.99cloud.net/html/2016/jiuzhouyuanchuang_1011/235.html)
https://www.youtube.com/watch?v=VsCYSZUOB6U
http://www.10tiao.com/html/362/201703/2654062885/1.html
>Neutron允许用户创建任意CIDR的子网，这造成了Neutron没有办法知道哪些子网是可以在一个路由空间的。Subnet pool的引入解决了地址重叠问题。
先理清概念区别：address scope，subnetpool，subnet.之间有如下关系：

![image.png](http://upload-images.jianshu.io/upload_images/3635313-9a50b9622e714c72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
一个 subnetpool 只能属于一个 address scope，相应的，在 subnetpool 中通过 address_scope_id 表示该 subnetpool 属于哪个 address scope，同时在 network 中通过 ipv4_address_scope 和 ipv6_address_scope 属性表示该 network 属于哪个 address scope。
```
#创建一个ipv4的 address scope,name is address-scope-ipv4
$neutron address-scope-create --shared address-scope-ip4 4
#在上述的address scope中创建一个subnetpool, prefixes 定义该 subnetpool 的大小，
#通过 default_prefixlen 定义在该 subnetpool 中创建 subnet 时的掩码（即上述示例中的 24）
$ neutron subnetpool-create --address-scope address-scope-ip4 \
  --shared --pool-prefix 203.0.113.0/21 --default-prefixlen 26 \
  subnet-pool-ip4
#create a subnet belong to the above subnetpool
$ neutron subnet-create --subnetpool subnet-pool-ip4 
```