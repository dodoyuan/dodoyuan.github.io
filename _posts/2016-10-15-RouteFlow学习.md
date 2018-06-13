---
layout:     post
title:      RouteFlow
subtitle:
date:       2016-10-15
author:     dodoYuan
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 网络技术
---

只针对论文学习内容进行总结，暂时不作深入研究
 
![overview](http://upload-images.jianshu.io/upload_images/3635313-22bdfd575f59cce1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
目前网络体系模型
 
![目前网络模型](http://upload-images.jianshu.io/upload_images/3635313-2649bb52ec7e487a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Openflow体系模型
 
![OF体系模型](http://upload-images.jianshu.io/upload_images/3635313-d0d8c942d25095eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  RouteFlow将IP路由协议和Open Flow网络融合到一起，是一个简单又功能强大的想法，为实现了openflow的转发设备提供控制平面，使其能够运行IP路由协议，如BGP、OSPF、IS-IS等。
   RouteFlow提供了一种高性价比的路由架构。RouteFlow项目主要有两个目标，第一个是提供一个可部署的将传统网络迁移到以控制器为中心的SDN的混合组网的解决方案。第二个是提供一个可用于研究IP路由和转发的平台。

Routeflow架构
![Routeflow架构](http://upload-images.jianshu.io/upload_images/3635313-9078ec2faf3ea1b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 
   如图，RF-controller类似一个顶层应用，运行在网络控制器（NC）上，NC负责通过南向协议如OF与底层物理网络的交换机进行通信，并为RF-control提供API接口，发现底层网络拓扑。
核心控制逻辑位于RF-server内，RF-server时刻监听相应的事件并保持全网的状态信息，当给一个物理OF交换机被发现，RF-server实例化一个虚拟机或选择一个提前实例化好的，每个虚拟机运行着开源的路由协议栈，并对应到底层OF交换机配置相应数量的网络接口，这些虚拟主机中的网络接口绑定到一个软交换机，通过软交换机可以动态控制连接关系。
一旦虚拟拓扑搭建完成，VM中的路由协议开始运行并且调整相应的FIBs，每个FIB的更新，RF-slave（slave demo）发送一个更新信息给RF-server，要求下发一个流表安装。
为了更好的信息交互，定义了一个简单的RouteFlow protocol，RF-protocol 消息可以是指令或者事件，可以看作一个升级版的OF协议，为了VM、RF-slave的配置和管理增加了一些新信息。
为了允许和传统网络的结合，RF使用流表项来匹配从物理网络传统设备中接收到的已知的路由协议信息，并将他们交给相应的VM，相反，在虚拟拓扑中生出的路由信息也将发送给底层物理网络。

要理解整个体系，需要明白以下问题：
1、映射。每个底层物理Openflow交换机都会映射到一个对应的虚拟的交换机，由RF-server维护，虚拟交换机之间组成一个虚拟的拓扑，运行传统的路由协议，生成FIB（forwarding information base）,并下发到对应的OF交换机，注意RF-server只为OF交换机生成一个虚拟映射。通过这种方式可以很好地与传统网络兼容。
有以下三种方式的映射：
 
2、虚拟化。以上三种结构的映射关系，实际上也是一种虚拟化思想，特别是后满两种，将一个虚拟主机对应到多台OF交换机可以大大减少底层网络细节，如不用每个节点之间维持复杂的路由消息，仅仅维持联通关系。从而只需在入接口和出接口运行传统路由协议即可。
3、OF 交换机需要映射到虚拟交换机根本原因是因为OF交换机不支持传统的路由协议，但要和传统兼容，即交换路由信息。可以每一个OF交换机都由控制器集中控制，比如邻居发来的hello包，根据相应格式构造一个reply包，但是当网络比较大时，通信量比较大，而且无法进行虚拟化。

但又有一个问题：这个架构实现了和传统网络的兼容，特别是不对现有路由协议做任何修改，但作为SDN体系的优势还需深入理解。

参考资料：
1、Nascimento M R, Rothenberg C E, Salvador M R, et al. Virtual routers as a service: the routeflow approach leveraging software-defined networks[C]//Proceedings of the 6th International Conference on Future Internet Technologies. ACM, 2011: 34-37. 
2、http://cpqd.github.io/RouteFlow/