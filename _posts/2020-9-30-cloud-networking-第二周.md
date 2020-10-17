---
layout:     post
title:      cloud Networking 第二周 
subtitle:   Host网络和CC
date:       2020-9-30
author:     haoran
header-img: img/post-bg-universe.jpg
catalog: true
tags: 
    - 云网络课程
    - 云网络基础
    - 路由
    - 网络虚拟化
---

> cloud networking 笔记


# 第二周内容
[课程链接](https://www.coursera.org/learn/cloud-networking/home/welcome)

非常值得上的一门课，导师所在的组2018年在NSDI发表了PCC-VIVACE以及2020的sigcomm 的PCC Proteus，上完以后对云网络相关知识可以有进一步的理解

## 摘要

主要涉及DC技术架构，交换机、路由器和拥塞控制
1、物理主机内部的网络。
2、TCP cc需要解决的问题。

## 2.1 HOST Virtualization

<img src="/img/cloudNetworkingClass/server-virtual.png"/>

主要介绍了HOST virtualization的现状

- 1、VM Base ：通过虚拟网卡进行

- 2、container base ：通过Namespace划分网络，通过NAT访问外部网络

<img src="/img/cloudNetworkingClass/docker-network.png"/>


## 问题： 如何处理数据包？

​		从网卡读数据包到CPU需要一定的开销，由于CPU不是专用的数据包处理器，单核的处理能力甚至无法超过10Gbps(不包括交换部分，只是从NIC到Userspace）。



At 100Gbps, with all packets being 100 Bytes in size (including the preamble, etc.), how much time can we afford to spend on processing each packet to be able to process packets at line-rate, sequentially?



*举个例子：某系统需要处理100Gbps的，每个packet  100Bytes，那么改系统需要的平均处理每个包的时间应当为8 ns*

*线速（line rate），指代指网络设备交换转发能力的一个标准，是交换机接口处理器或接口卡和数据总线间所能吞吐的最大数据量。*

​		而虚拟交换机的引入导致这部分开销变的更大（因为不仅需要考虑数据包读取，还需要考虑转发，这导致了CPU资源的浪费和时延的增加）

### 关键问题：如何提升虚拟机网络通信的性能（降低不必要的CPU开销）

**方案1 特定的硬件(hardware approach)**

最简单的思路：给VM访问NIC的权限。（因为NIC是专门用于处理数据包的，而CPU不是）

进一步的问题：如何分配NIC的资源给VM?

常用的办法：SR-IOV（single-root I/O virtualization）

<img src="/img/cloudNetworkingClass/SR-IOV.png"/>

图中的pNIC是一张支持硬件虚拟化网卡，网卡可以提供一些简单诸如转发的虚拟功能，此时，通过将VM和VF的映射，实现VM对网卡资源的直接DMA(direct memory access）访问。pNic通过一个简单的L2 switch来实现流量的转发。

优点：bypass的，虚拟化层只需要参与连接关系的确立和网卡资源的分配，不需要参与数据包处理，节省了CPU资源的同时提高了处理性能。

缺点：变得不灵活
1、VM和网卡绑定。
2、二层的转发并不灵活。（不能用通用的转发规则实现）

**方案2 软件的办法（software approach)/SDN的办法**

优点：更灵活

案例：OpenVswitch

<img src="/img/cloudNetworkingClass/ovs-framework.png"/>

- OVS**同时运行**在Userspace和kernel space

- **userspace存储流表信息（状态）**，当一个包到达时，如果kernel部分没有匹配表，将通过vswitchd进行查询，同时，将在内核部分增加一个**hash表来转发**之后来的相同五元组的数据包。

对比两种方案，分别适用于两种不同的场景：

1、硬件的方案适用于要求线速，尽可能最大利用网卡性能，降低CPU的处理开销，但是缺点是不够灵活

2、软件方案通过OVS,ovs通过user space 和 kernel space结合的方式，尽可能提高了软件交换的处理能力。


## 总结 

虚拟化使得网络进入了主机中，一个重要的议题就是如何权衡灵活性和性能的问题

灵活性的体现：希望能快速迁移服务，同时对业务流量进行处理.例子 OvS

性能的体现：希望能尽可能利用网卡资源。例子 SR-IOV

但是一些场景下可以利用两者的结合来完成(FPGA等等。。）



## 2.2 Routing 

- 问题描述：解决如何寻找网络中src 到 dst 的路径

- 特点：分布式的算法，同时需要避免环路

**运行在二层网络的转发问题：**

早期的设计是生成树协议（STP）针对的是spining tree结构的DC网络，这种特点的拓扑在小规模时运行能达到较好的效果 

<img src="/img/cloudNetworkingClass/image-20201002215633255.png"/>


为了保证业务的正常运转，提高鲁棒性，通常会出现多条冗余链路，如下图所示

<img src="/img/cloudNetworkingClass/image-20201002220123402.png"/>

此时的网络条件下，生成树协议的效果会变得十分糟糕，此外生成树的问题在于，只能使用单条路径，这使得网络的鲁棒性非常差（一条链路中断就无法到达）

此外，生成树协议依赖**广播进行**，如果存在环路，则可能造成**广播风暴**的问题。

有许多的解决办法，

#### Trill's design （一种替代STP的办法，提供了解决问题的思路）

特点：在交换机中运行了一种link state protocol链路状态信息，这样子每个交换机可以获得网络的拓扑。每个交换机可以理解如何到达其他的交换机。

<img src="/img/cloudNetworkingClass/image-20201002221141118.png"/>


解决的关键问题：

- 每个链路都是可用的，可以实现multi-path routin。

缺点：

- 需要更新设备。



### 2.2.2另一种思路： 基于IP层的解决办法

#### 2.2.2.1 OSPF


<img src="/img/cloudNetworkingClass/image-20201002223030210.png"/>


OSPF

[OSPF知识点复习](https://zhuanlan.zhihu.com/p/41341540)

[华为的OSPF文档](https://support.huawei.com/enterprise/zh/doc/EDOC1100082073)

过程：邻居发现、路由交换、路由计算、路由维护\

维护的表：

1、 邻居列表：列出每台路由器全部已经建立邻接关系的邻居路由器
2、 链路状态数据库：列出网络中其他路由器的信息，由此显示了全网的网络拓扑
3、 路由表：列出通过SPF算法计算出到达每个相连网络的最佳路径

特点：

- 基于链路信息的最短路径协议
- 在子网中运行，不考虑域外的问题

优点：

- 和Trill类似，是一哥维持链路状态的协议，同时不需要修改底层的设备

缺点：

- 需要运行在三层，因此需要对网络进行配置，包括子网等等，因此不是一个即插即用的协议。

接着讨论 扩展性的问题，OSPF的缺点在于路径的选择自由度较低（最短路径优先），这可能导致一些链路没有被很好的利用，即使在DC中用到了ECMP。为数不多的办法是通过修改OSPF的链路权重，来避免此类的情况的发生、

### 2.2.2.2 BGP


<img src="/img/cloudNetworkingClass/image-20201002225537359.png"/>


BGP又可以分为IBGP（Interior BGP ：同一个AS之间的连接）和EBGP（Exterior BGP：不同AS之间的BGP连接） 

[BGP资料](https://zhuanlan.zhihu.com/p/25433049)

备注：AS（Autonomous system）自治系统，指在一个（有时是多个）组织管辖下的所有IP网络和路由器的全体，它们对互联网执行共同的路由策略。

特点：

- 针对链路状态和可达性进行维护（不强求最短路径）
- 利用邻居信息进行传输（path vector protocol）

<img src="/img/cloudNetworkingClass/image-20201002225643329.png"/>

优点：

- 基于三层的，因此避免了二层应用的可扩展性问题

缺点：

- 相比二层的协议，如果需要迁移虚拟机，问题会变得更加复杂。（DC中的问题）
- 收敛慢
- 过于灵活也会导致配置变得更加复杂



OSPF是基于链路状况计算路由的；

BGP本身不会去计算路由，只会把其他协议生成的路由拿来用;

一个是生产路由的，一个是玩路由的。OSPF生产的方式很精密，保证无环路，但多业务支撑不行；BGP不生产，只做调度使用，所以业务支撑好，扩展属性让路由规划多了很多选择。

总的来说，当一个网络拓扑较小的时候，求解路径通常是比较简单的问题，但是随着拓扑的变大，即使设计之初采用了leaf-spine的结构，加上了冗余链路等，问题都会随之发生变化。



下一章：讨论多路径传输的问题



### 2.2.3 多路径传输问题

<img src="/img/cloudNetworkingClass/image-20201004222940904.png"/>


http://ccr.sigcomm.org/online/files/p63-alfares.pdf

问题：多路径传输时，如何保证尽可能的减少拥塞

如图中的p-fat-tree的组网，N口的交换机，可以有
$$
(N/2)^2
$$
条传输路径。



优化目标：minimize congested

难点：

- 大量的数据流都是短流，导致网络的状态变化非常迅速
- 集中式的流量调度/负载均衡可能导致很高的计算开销
- 需要一种分布式的解决方案，可以更快的收集信息，从而避免之前的问题发生。

主流的办法

- 随机发送

- ECMP
  
  
  
  

ECMP带来的问题：

- TCP乱序---> 会被错误地认为是丢包---->速率下降
- 解决办法：同一个五元组的用同一条路径进行传输

带来的新的问题：

- 可能导致负载不均衡
- ECMP的决策只是在本地的，无法意识到错误决策带来的结果



问题1的解决方案：

- 基于flowlet的负载均衡方案 （用于替代基于packet的方案和基于流的方案）
- flowlet 特点：两个packet的间隔足够长，不会导致乱序的情况发生，因此被认定为是

问题2的解决方案：

- conga 选择最低的path （一些细节的设计，例如测量的时候不再采用频繁发送探针的形式，而是将拥塞信息添加到数据包的包头上）



### 2.3 拥塞控制

#### 2.3.1 basic  idea

<img src="/img/cloudNetworkingClass/image-20201005173127203.png"/>


#### 2.3.2 Problem

keyword：high lentency and TCP incast

传统TCP （reno tahoe）

- buffer filling --- lead to long delay
- 丢包不是一个很好的判断机制 --- 未必是队列满了才发生丢包
- Mulitplcative decrease can be too agressive，perhaps sender's rate is only larger than the available capacity
- 相比传播时延，传输和排队时延占到了主要的部分。(距离300m的传播时延只有1.5 microsecond 1.5 us）

DC 中的特定问题 TCP-INCAST：

<img src="/img/cloudNetworkingClass/image-20201006215446553.png"/>

e.g.  Web 搜索通常需要对多个服务器/数据库发送请求来获得数据，这种业务通常对时延非常敏感，但是这种多对一的通信模式也可能出现诸多问题

Incast 是指一种多对一的通信模式，常见于部署了大量的分布式存储或者计算服务（如Hadoop, MapReduce, HDFS, Cassandra等）的数据中心中。当然，Incast 也可以特指 TCP Incast，因为上述这些服务大部分都是基于 TCP 的。

Incast 发生在一个 client 同时向后端的多个 server 节点发起数据请求，在某些情况下，server 集群会同时响应这些请求，那么在**某一段时间内会出大量的服务节点会向同一个机器发送数据的情况**，从而在短时间内使得一个服务器接收的流量激增。由于应用类型的不同，Incast 模式中的数据流可能存在的时间会非常短，例如，client 向40个 server 节点各发送一个数据请求，假定每个请求会回应一个2KB的数据，那么每个节点实际上只回应了2个IP包。

TCP Incast 也用于描述由 Incast 通信模式导致的网络拥塞。**短时内的流量激增会导致接收服务器的所对应的交换机的出口处发生拥塞，交换机的出端口的缓存溢出，使得部分数据包被丢弃**，而发送服务器感知到丢包之后需要重传之前的数据包，最终导致单个 TCP 流的报文吞吐增长缓慢。另外，接收服务器上的应用可能需要等到接收完全部的数据，或者因为超时的原因只是返回部分数据，无论是以上何种方式，服务器的速度、质量以及稳定性都受到了影响。更加不幸的是由 Incast 引起的拥塞能很容易影响这类应用和使用这些应用的用户体验

Suppose that to address TCP incast, we reduce the TCP connection timeout value. Which of the following is true?

<img src="/img/cloudNetworkingClass/image-20201006220123761.png"/>

根据测量的结果数据可以看出，随着发送请求的server的增加，大量的incast导致了整体的吞吐下降。

#### some method

confict：

- high thoughput ：需要用buffer来缓存burst traffic 
- low latency  ：过大的buffer可能导致更长的排队时延

案例:
<img src="/img/cloudNetworkingClass/image-20201006222720940.png"/>

DCTCP

- 思想：提前调整速率而不是不再等待buffer filling丢包（此时的延迟可能已经很大）
- 策略：**ECN机制**，设置两个阈值，当平均队列高于High thresh时交换机给数据包添加一个拥塞的bit，当队列小于low thresh时，不再标记拥塞。当接收端收到带有拥塞标记的数据包时，回传一个相应的数据包。发送方收到这个拥塞标记时，提前调整速率
- 相比基于丢包的机制，这种办法可以提前获取队列信息，避免丢包（更早获得一个减速的信号）

<img src="/img/cloudNetworkingClass/image-20201006223159071.png"/>

方案关键：

- 用ECN来标记超过阈值的数据包

- 接收端根据一组的ack stream中的ECN包的数据，以及此时正在传输的流的数量调整速率，而不是TCP reno的乘性减这种agressive的办法。
  $$
  cwnd = (1-α/2)*cwnd
  $$
  其中 α和ECN包的比重，此时同时存在的流的条数等参数相关

- 可以维持较低的buffer占用。
<img src="/img/cloudNetworkingClass/image-20201011223708199.png"/>
<img src="/img/cloudNetworkingClass/image-20201011223114851.png"/>


<img src="/img/cloudNetworkingClass/image-20201011224113863.png"/>




RCP，主要解释了乘性增减的问题。

What considerations govern the setting of DCTCP’s congestion marking threshold, *K*?



补充：

DCQCN

BBR