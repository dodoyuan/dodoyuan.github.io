---
layout:     post
title:      openstack安装小记
subtitle:
date:       2017-5-10
author:     dodoYuan
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - openstack
---

openstack安装一般有两种方式，Devstack单节点安装和多节点安装。单节点适合自己随便玩玩，认真学还是推荐多节点。
Devstack安装  ：https://my.oschina.net/zyzzy/blog/74088
本笔记针对的是openstack Newton版本
#### 1 资料：
官网的资料是比较基础的：
https://docs.openstack.org/newton/install-guide-ubuntu/keystone-verify.html 

#### 2 问题备注：
1、 KVM虚拟化可能出现一点问题
![image.png](http://upload-images.jianshu.io/upload_images/3635313-5c7170fefd4d0182.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2、配置中会有涉及到IP，因为IP会经常变动，要引起注意。
* 第一个出现的也是主要的，在 /etc/hosts  文件中, controller 指定到一个IP地址
* 第二个出现在 /etc/nova/nova.conf 中，涉及到一个固定IP
还有几个，根据官网以及自己配置过程，可能是以后问题的一个来源。

#### 3 踩过的坑
##### (1)安装dashboard界面打不开，最后超时
https://docs.openstack.org/newton/install-guide-ubuntu/horizon-verify.html
######最后参考[文章](http://www.aboutyun.com/home.php?mod=space&uid=1310&do=blog&quickforward=1&id=3126) 
>在配置文件文件/etc/apache2/conf-available/openstack-dashboard.conf中首行添加： WSGIApplicationGroup %{GLOBAL}


##### (2) 能进入登录界面，但是输入用户名密码后显示  “something went wrong”
首要一点就是，出现问题要找日志文件。对web服务，则查看 /var/log/apache2/error.log文件，出现这个有很多情况
>RuntimeError: Unable to create a new session key. It is likely that the cache is unavailable
解决：在 /etc/memcached.conf文件中，将 -l 10.0.0.11 改为 -l 0.0.0.0 
错误日志已经告诉可能原因是cache问题。做了修改后一定要重启服务。
##### `service memcached restart`

还有常见的问题就是URLendpoint服务出现问题，需要去确保之前的操作[添加用户]( https://docs.openstack.org/newton/install-guide-ubuntu/keystone-users.html)没有问题，以及以防万一，确保galance中也没问题。

#### (3)Unable to establish connection to http://controller:35357/v3/auth/tokens
>Discovering versions from the identity service failed when creating the password plugin. Attempting to determine version from URL.
Unable to establish connection to http://controller:35357/v3/auth/tokens

这个问题算数很莫名其妙，进行多次试验后，重启Ubuntu后出现了以上问题，往回排查，最终确定在keystone验证功能出问题了，
唯一可能的就是数据库，删除掉之间建立的 keystone数据库，重新生成。问题可以得以解决。
`mysql>DROP DATABASE keystone;`
继而又出现了另一个问题
>public endpoint for network service not found
解决：可能是重建keystone数据库后造成影响，重新配置主要有以下,需要将之前建立的数据库重新配置。
[配置1](https://docs.openstack.org/newton/install-guide-ubuntu/glance-install.html)
[配置2](
https://docs.openstack.org/newton/install-guide-ubuntu/neutron-controller-install.html)
这个问题很奇怪，因为后面重启系统后又没有出现，运行验证如openstack compute service list等都能正常工作。
总结原因，一是要操作规范，自己在安装时没有更行操作系统，一定要运行 
### ` apt update && apt dist-upgrade`

控制节点和计算节点都需要两个网卡，这个对自己来说应该很熟悉，主要是OVS相关，一个网卡作为终端管理，一个用于数据转发。
在控制节点中目前是使用ens33网卡作为绑定到OVS上的网卡，不分配IP，而在计算节点中使用的是外接网卡完成。
若要修改，在 /etc/neutron/plugins/ml2/linux....中。

##### (4)一些注意
开机重启后，有些服务可能不会自动打开，比如数据库 mysql以及memcache，memcache是个缓存加速，没有启动会造成整个系统运行十分慢。

最终成功打开 dashboard界面。
![image.png](http://upload-images.jianshu.io/upload_images/3635313-c2eacb746bf807b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)