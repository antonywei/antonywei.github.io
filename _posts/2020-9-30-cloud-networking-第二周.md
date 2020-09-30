---
layout:     post
title:      cloud Networking 第二周 Tagged Pointer
subtitle:   浅谈 Tagged Pointer
date:       2020-9-30
author:     BY
header-img: img/post-bg-universe.jpg
catalog: true
tags: 
	-cloud networking class
---
# 第二周内容
## 摘要
主要涉及DC技术架构，交换机、路由器和拥塞控制
1、物理主机内部的网络。
2、TCP cc需要解决的问题。

## HOST Virtualization

![avatar](cloudNetworkingClass\server-virtual.png)

主要介绍了HOST virtualization的现状

- 1、VM Base ：通过虚拟网卡进行

- 2、container base ：通过Namespace划分网络，通过NAT访问外部网络

![avatar](cloudNetworkingClass\docker-network.png)


## 问题： 如何处理数据包？

从网卡读数据包到CPU需要一定的开销，由于CPU不是专用的数据包处理器，单核的处理能力甚至无法超过10Gbps(不包括交换部分，只是从NIC到Userspace）。

而虚拟交换机的引入导致这部分开销变的更大（因为不仅需要考虑数据包读取，还需要考虑转发，这导致了CPU资源的浪费）

### 关键问题：如何提升虚拟机网络通信的性能（降低不必要的CPU开销）

**方案1 特定的硬件(hardware approach)**

最简单的思路：给VM访问NIC的权限。（因为NIC是专门用于处理数据包的，而CPU不是）

进一步的问题：如何分配NIC的资源给VM?

常用的办法：SR-IOV（single-root I/O virtualization）

![avatar](cloudNetworkingClass\SR-IOV.png)

图中的pNIC是一张支持硬件虚拟化网卡，网卡可以提供一些简单诸如转发的虚拟功能，此时，通过将VM和VF的映射，实现VM对网卡资源的直接DMA(direct memory access）访问。pNic通过一个简单的L2 switch来实现流量的转发。

优点：bypass的，虚拟化层只需要参与连接关系的确立和网卡资源的分配，不需要参与数据包处理，节省了CPU资源的同时提高了处理性能。

缺点：变得不灵活
1、VM和网卡绑定。
2、二层的转发并不灵活。（不能用通用的转发规则实现）


**方案2 软件的办法（software approach)/SDN的办法**

优点：更灵活

案例：OpenVswitch

![avatar](cloudNetworkingClass\ovs-framework.png)

OVS同时运行在Userspace和kernel space

userspace存储流表信息，当一个包到达时，如果kernel部分没有匹配表，将通过vswitchd进行查询，同时，将在内核部分增加一个hash表来转发之后来的相同五元组的数据包。

对比两种方案，分别适用于两种不同的场景：

1、硬件的方案适用于要求线速，尽可能最大利用网卡性能，降低CPU的处理开销，但是缺点是不够灵活

2、软件方案通过OVS,ovs通过user space 和 kernel space结合的方式，尽可能提高了软件交换的处理能力。


## 总结 

虚拟化使得网络进入了主机中，一个重要的议题就是如何权衡灵活性和性能的问题

灵活性的体现：希望能快速迁移服务，同时对业务流量进行处理.例子 OvS

性能的体现：希望能尽可能利用网卡资源。例子 SR-IOV

但是一些场景下可以利用两者的结合来完成(FPGA等等。。）