---
layout:     post
title:      Segment Routing基础
subtitle:
date:       2016-10-15
author:     dodoYuan
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 网络技术
---

##### 目前网络中的局限：
* RSVP-TE多用于广域网流量工程，通过逐条建立路径，每个节点都需要维护大量的状态信息，会受限于规模，不支持大规模部署；同时RSVP-TE不支持等价路径分担，使网络利用率不高。
+ 网络中应用和网络离的很远，即网络中数据传送常常是不区分服务的（待深入理解）

SR可以有效解决上面两个问题，一是通过segment list确定路径信息，中间节点完全不需要维持路径状态信息，只负责转发，适合大规模部署；二是SR是实在的应用驱动网络，应用提出需求，如带宽、时延等，控制器基于全局拓扑，如链路状态、连路利用率信息计算路径，并映射到segment list所定义的路径上。
##### SR 体系结构
整个体系上，SR采用了类似于SDN架构：数据平面和控制平面相分离。数据平面定义了 如何将segment应用于数据包上，即节点如何根据segment处理一个数据包。控制平面定义了segment 标识如何在节点之间通告，一般通过内部网关协议（IGP）的链路状态信息，这就要求对现有网络路由协议（IS-IS、OSPF）进行适当修改，已经有相应的文献进行了研究。同时控制平面也处理SR路径的计算。
SR实现了源路由和隧道模式
###### Segment知识点：
*	Segment ：任意类型指令的一种标识。不能局限于MPLS所说的标签，要看所承载的网络和功能，若运行于MPLS网络，segment语义上为标签；若运行于IPv6网络，语义上为IPv6地址。
-	Segment分为全局和本地，对全局segment，全局可见，全局有效，而本地segment全局可见，本地有效。
-	另外一个概念：SRGB，预留出来的全局标签范围，默认为16000-23999，全局segment在SRGB中取值。通告全局segment，先通告SRGB，再加一个索引，如通告SRGB范围为 16000-23999，索引为1，则生成的全局segment为16001。注意一个区域内必须保证节点通告的SRGB一致。

##### SR基本工作原理：
内部网关协议中使用 prefix-SID 的几种用法
###### 1. 全局SID使用
 
![全局SID示例](http://upload-images.jianshu.io/upload_images/3635313-a2eb4a8ae555829c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图为SR转发示意图，假设A点要发送数据包到Z点，会经过以下几个过程，首先，底层网络节点交换node SID（segment ID），控制器或者决策中心根据应用需求（带宽、时延等）结合生成一个segment list，压入数据包。上图中，到Z的数据包压入了三个SID，首先第一个SID 16072 为active状态，说明数据先转发到SID 为16072的节点C，路由生成依赖于原有的IP路由，可以负载均衡（ECMP）。到C后发现第一个SID为自己，则弹出，置下一个SID 29003为active状态，以此类推，最终到达节点Z。
一个支持SR的节点（交换机或路由器等）必须支持以下三种操作：
*	CONTINUE：基于active SID 的转发行为
*	PUSH：添加一个segment ID 到数据包头部，并将该SID置为active 状态。
*	NEXT：将下一个SID置为active 状态。
###### 2. 局部SID与全局SID的结合
 
![结合示例](http://upload-images.jianshu.io/upload_images/3635313-ffcb5ba4de156ee0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中24045为局部SID，在本区域内有效，每个节点维持了与邻接节点之间的 邻接SID，通常用于 节点之间有多条路径时，指定路径。例如节点 4 到节点5有两条链路，则每条链路指定一个局部SID。
###### 3. 实现负载均衡
 
![负载均衡](http://upload-images.jianshu.io/upload_images/3635313-2ebd2a951b8d9144.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中 PE1和PE2拥有自己的全局SID外，还向外通告了一个共同的SID 16100，则节点A到16100可以进行多路负载均衡的同时还能相互形成冗余，提高网络的抗毁性。
###### SR应用
可以广泛应用于流量工程、网络冗余配置等等。
##### 总结：
基于Openflow协议的SDN架构对现有网络改动太大，如需要交换机支持openflow协议，同时上层架构如路由协议也要进行改动以支持控制器集中控制。虽然不同于openflow在整个SDN架构中为南向接口协议这一地位，SR则提供了一种更有效的演变方式，甚至可以完成在现有网络架构中支持转发与控制平面分离而不需要openflow协议参与，具有很强大的灵活性。


参考资料：
1、Filsfils C, Nainar N K, Pignataro C, et al. The segment routing architecture[C]//Global Communications Conference (GLOBECOM), 2015 IEEE. IEEE, 2015: 1-6. 
2、http://www.jianshu.com/p/03653ef39d0d