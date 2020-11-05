---
 Nlayout:     post
title:      【网课】cloud Networking 笔记：虚拟网络 NVP of vmware
subtitle:   SDN网络
date:       2020-10-28
author:     haoran
header-img: img/post-bg-universe.jpg
catalog: true
tags: 
    - 云网络课程
    - SDN
    - 虚拟网络
typora-root-url: ..
---

## VMware Network virtualization platform(NVP)

overview:

将硬件平面和软件平面分离，因此可以将软件平面的资源指定分配给软件平面，包括虚拟化CPU,memory，disk以及网络IO.通过虚拟化资源，平台可以和多个虚拟化应用配合使用，例如防火墙，路由，web filters，以及IPS等。这些功能逻辑上是一个独立的硬件的应用，但是可以用同一个硬件设备来装载。这个技术的关键优点在于通过网络应用保证了网络的独立运行，同时具备了动态的资源管理和服务管理功能。

设计需求

1. Service：Arbitrary network topology

![image-20201028162758922](/img/cloudNetworkingClass/2020-10-28-cloud-networking-NVP/image-20201028162758922.png)

相比VL2中将物理层和虚拟层分离，NVP更清晰地建立一个虚拟机网络层，将任意的网络服务和物理网络分离。



![image-20201028205857844](/img/cloudNetworkingClass/2020-10-28-cloud-networking-NVP/image-20201028205857844.png)

 ### 一个NVP系统的建立

![image-20201028210344077](/img/cloudNetworkingClass/2020-10-28-cloud-networking-NVP/image-20201028210344077.png)

1. 搭建物理网络 （NVP视角下则是一堆主机通过IP接入网络）
2. 一部分主机运行着租户的VM
3. 租户设计了一个L2-L3-L2的网络
4. 控制器通过API配置相应的服务和网络信息
5. underlying  software switching data plane which is at the virtualization layer of each of those servers on the tenant's VM resides





Challenge: Performance

1. Large amount of state to compute
2. 
3.  
4. 