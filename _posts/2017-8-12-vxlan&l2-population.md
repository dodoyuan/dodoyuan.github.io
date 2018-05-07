---
layout:     post
title:      vxlan 和 L2 population
subtitle:
date:       2017-8-12
author:     dodoYuan
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 云网络
---

### vxlan
#### 1、vxlan是什么
vxlan-虚拟扩展局域网，是现代数据中心的的一种网络虚拟化技术，即在传统的三层IP网络上虚拟出一张二层的叠加网络，本质上是一种大二层的虚拟网络技术，引入一个UDP格式封装的外层隧道作为数据的链路层，原有数据报文内容作为隧道净负荷来传输，使得净荷数据可以轻松在二三层网络中传播。
#### 2、为什么要vxlan
一种新技术的出现，肯定是传统的技术遇到了瓶颈，vxlan的出现主要是想解决以下几个问题：
1、STP协议和VLAN的限制。
* STP解决二层帧循环转发问题，通过在二层网络之间构造一棵生成树来阻塞冗余路径，从而不支持多路径，而数据中心网络或大型网络中多路径是很常见，用来提高系统吞吐量。vxlan可以有效利用多路径，因为vxlan是在三层网络上构建一个二层网络，所以只要三层网络路由协议支持多路径，vxlan就支持多路径；
* VLAN支持将二层网络划分为最大4094个虚拟局域网（0和4095保留，VLAN ID只有12位），在虚拟化、STP限制以及多租户环境等原因影响下，VLAN不能满足；

2、多租户环境。
* 公有云服务为多租户提供服务，采用二层或者三层网络隔离租户网络流量。二层网络使用VLAN，在租户过多时，VLAN不够用；三层网络隔离方案因为多用户使用相同的地址集合，需要云服务商提供特有的隔离方法，例如要求所有租户使用IP隔离网络而不是二层，或者使用非IP的3层协议用于内部虚拟机之间通信；

3、ToR交换机MAC地址表大小的限制。
* 典型的ToR交换机可以连接24-48个物理服务器，由于虚拟化，单个物理服务器将有多个虚拟机，实际上等于连接更多的服务器，容易造成MAC表溢出，如果溢出，交换机停止学习新地址直到MAC表有空间；

4、虚拟机迁移的需要。
* 为保证业务的高可用性，需要迁移虚拟机到新的服务器上，例如跨数据中心，此时需要保证虚拟机IP地址和MAC地址不变，需要业务网络是一个二层网络，并且网络本身具备多路径的冗余备份和可靠性。
#### 3、vxlan协议封装格式
![image.png](https://upload-images.jianshu.io/upload_images/3635313-530a90790066344f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
vxlan是在三层网络上封装二层协议，在原始数据包上增加了50个字节的封装头部。首先是VXLAN头部，8字节，大部分是保留字段，有24个比特（缩写为VNI）用来标识 individual VXLAN overlay network ，传输层协议使用UDP，源端口号是4789，也就是说开启了vxlan的路由器，会在端口4789上启动一个进程，用来处理接收到的数据包，从而知道如何对数据进行解封装。再外层是IP头部，实现传统的三层路由，IP是VTEP相关的IP地址，最外面是MAC地址，对应VTEP的二层地址。
**VTEP是什么**
![image.png](https://upload-images.jianshu.io/upload_images/3635313-e6b4f0e13cff227b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上，VTEP主要实现对数据包进行封装和解封装，同时还会进行mac到VTEP IP的地址学习。
>VXLAN uses VXLAN tunnel endpoint (VTEP) devices to map tenants end devices to VXLAN segments and to perform VXLAN encapsulation and de‐encapsulation. Each VTEP function has two interfaces.The IP interface has a unique IP address that identifies the VTEP device on the transport IP network known as the infrastructure VLAN. The VTEP device uses this IP address to encapsulate Ethernet frames and transmits the encapsulated packets to the transport network through the IP interface. A VTEP
device also discovers the remote VTEPs for its VXLAN segments and learns remote MAC Address‐to‐VTEP mappings through its IP interface. The functional components of VTEPs and the logical topology that is created for Layer 2 connectivity across the transport IP network is shown in this Figure.

#### 4、数据流图
![image.png](https://upload-images.jianshu.io/upload_images/3635313-6c129b35382e8410.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>In this Figure, Host‐A and Host‐B in VXLAN segment 10 communicate with each other through the VXLAN tunnel between VTEP‐1 and VTEP‐2. This example assumes that address learning has been done on both sides, and corresponding MAC‐to‐VTEP mappings exist on both VTEPs. When Host‐A sends traffic to Host‐B, it forms Ethernet frames with MAC‐B address of Host‐B as the destination MAC address and sends them out to VTEP‐1. VTEP‐1, with a mapping of MAC‐B to VTEP‐2 in its mapping table, performs VXLAN encapsulation on the packets by adding VXLAN, UDP, and outer IP address header to it. In the outer IP address header, the source IP address is the IP address of VTEP‐1, and the destination IP address is the IP address of VTEP‐2. VTEP‐1 then performs an IP address lookup for the IP address of VTEP‐2 to resolve the next hop in the transit network and subsequently uses the MAC address of the next‐hop device to further encapsulate the packets in an Ethernet frame to send to the next‐hop device.
    The packets are routed toward VTEP‐2 through the transport network based on their outer IP address header, which has the IP address of VTEP‐2 as the destination address. After VTEP‐2 receives the packets, it strips off the outer Ethernet, IP, UDP, and VXLAN headers, and forwards the packets to Host‐B, based on the original destination MAC address in the Ethernet frame.

### l2 population
#### 1、L2 Population 原理
L2 Population 是用来提高 VXLAN 网络 Scalability 的。
通常我们说某个系统的 Scalability 好，其意思是： 当系统的规模变大时，仍然能够高效地工作。

下图是一个包含 5 个节点的 VXLAN 网络，每个节点上运行了若干 VM。
现在假设 Host 1 上的 VM A 想与 Host 4 上的 VM G 通信。VM A 要做的第一步是获知 VM G 的 MAC 地址。于是 VM A 需要在整个 VXLAN 网络中广播 APR 报文：“VM G 的 MAC 地址是多少？”

![image](http://upload-images.jianshu.io/upload_images/3635313-5d1bd2dcd7f330bc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161115-1479162754573009621.png")

如果 VXLAN 网络的节点很多，广播的成本会很大，这样 Scalability 就成问题了。幸好 L2 Population 出现了。

![image](http://upload-images.jianshu.io/upload_images/3635313-a7b7268e08f26914.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161115-1479162754747078834.png")

L2 Population 的作用是在 VTEP 上提供 Porxy ARP 功能，使得 VTEP 能够预先获知 VXLAN 网络中如下信息：
1\. VM IP -- MAC 对应关系
2\. VM -- VTEP 的对应关系

当 VM A 需要与 VM G 通信时：
1\. Host 1 上的 VTEP 直接响应 VM A 的 APR 请求，告之 VM G 的 MAC 地址。
2\. 因为 Host 1 上的 VTEP 知道 VM G 位于 Host 4，会将封装好的 VXLAN 数据包直接发送给 Host 4 的 VTEP。

这样就解决了 MAC 地址学习和 APR 广播的问题，从而保证了 VXLAN 的 Scalability。

那么下一个关键问题是：
**VTEP 是如何提前获知 IP -- MAC -- VTEP 相关信息的呢**？

答案是：
1.  Neutron 知道每一个 port 的状态和信息； port 保存了 IP，MAC 相关数据。

2.  instance 启动时，其 port 状态变化过程为：down -> build -> active。

3.  每当 port 状态发生变化时，Neutron 都会通过 RPC 消息通知各节点上的 Neutron agent，使得 VTEP 能够更新 VM 和 port 的相关信息。

4.  VTEP 可以根据这些信息判断出其他 Host 上都有哪些 VM，以及它们的 MAC 地址，这样就能直接与之通信，从而避免了不必要的隧道连接和广播。

理解了工作原理，下节我们学习如何在 Neutorn 中配置 L2 Population。
#### 2、l2 population 配置
前面我们学习了L2 Population 的原理，今天讨论如何在 Neutron 中配置和启用此特性。

目前 L2 Population 支持 VXLAN with Linux bridge 和 VXLAN/GRE with OVS。

可以通过以下配置启用 L2 Population。

在 /etc/neutron/plugins/ml2/ml2_conf.ini 设置 l2population mechanism driver。

![image](http://upload-images.jianshu.io/upload_images/3635313-46f493ea9680b856.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161117-1479336871744014127.png")

> mechanism_drivers = linuxbridge,l2population

同时在 [VXLAN] 中配置 enable L2 Population。

![image](http://upload-images.jianshu.io/upload_images/3635313-32f3ec810bfc729d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161117-1479336871775097441.png")

L2 Population 生效后，创建的 vxlan-100 会多一个 Proxy ARP 功能。

![image](http://upload-images.jianshu.io/upload_images/3635313-18bf06571f3194fa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161117-1479336871845026357.png")

查看控制节点上的 forwarding database，可以看到 VTEP 保存了 cirros-vm2 的 port 信息。

![image](http://upload-images.jianshu.io/upload_images/3635313-0933f955fa645e3c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161117-1479336871883017370.png")

cirros-vm2 的 MAC 为 fa:16:3e:1d:23:a3。 VTEP IP 为 166.66.16.11。

当需要与 cirros-vm2 通信时，控制节点 VTEP 166.66.16.10 会将封装好的 VXLAN 数据包直接发送给计算节点的 VTEP 166.66.16.11。

我们再查看一下计算节点上的 forwarding database：

![image](http://upload-images.jianshu.io/upload_images/3635313-be60323b69bc8e95.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "http://7xo6kd.com1.z0.glb.clouddn.com/upload-ueditor-image-20161117-1479336871950004438.png")

fdb 中保存了 cirros-vm1 和 dhcp 的 port 信息。 当需要与它们通信时，计算节点 VTEP 知道应该将数据包直接发送给控制节点的 VTEP。


### 参考

https://networkop.co.uk/blog/2016/05/06/neutron-l2pop/
[l2 population 配置](https://www.ibm.com/developerworks/community/blogs/132cfa78-44b0-4376-85d0-d3096cd30d3f/entry/%E9%85%8D%E7%BD%AE_L2_Population_%E6%AF%8F%E5%A4%A95%E5%88%86%E9%92%9F%E7%8E%A9%E8%BD%AC_OpenStack_114?lang=en_us)