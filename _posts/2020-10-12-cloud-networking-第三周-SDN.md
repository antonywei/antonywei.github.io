---
layout:     post
title:      cloud Networking 第三周笔记： SDN网络
subtitle:   SDN网络
date:       2020-10-12
author:     haoran
header-img: img/post-bg-universe.jpg
catalog: true
tags: 
    - 云网络课程
    - SDN
typora-root-url: ..
---

> cloud networking 笔记

# Overview

- SDN架构。
- 多租户网络的DC的需求和挑战。
- 多租户网络如何通过SDN技术满足需求



## 3.1.1 SDN architecture

<img src="/img/cloudNetworkingClass/image-20201012204550495.png"/>



网络的问题（大方向上的）：

- 复杂的、分布式带来的问题。
- 不存在直接的、多元的控制方式。
- 不同厂商的硬件是相互独立的。



新的目标

- 将转发和控制分离<img src="/img/cloudNetworkingClass/image-20201012205043820.png"/>

<img src="/img/cloudNetworkingClass/image-20201012205326292.png"/>




一个简单的NOX控制器的例子，Match - action -安装规则

匹配数据包、执行相应的动作，拓扑发现，监控等。这在传统设备上是很难实现的。



SDN 的发展历史

Label switch / MPLS (1997) 

- tag switching architecture overview
- set up explicit path for classes of traffic 
- 基于label的决策，而不需要重新解析ip报文（在MPLS网络中生效）

Active Network （1999）

- Packet header carries（pointer to）program code
- 在包头嵌入一段代码/指令， 让路由器去执行。

Routing Control Platform（2005）

- centeralize computation of BGP， push to border routers via iBGP
<img src="/img/cloudNetworkingClass/image-20201017130157810.png"/>


Ethane（2007）

- allow program policy for enterprise（access control in a centralized way and have it then pushed out to the entire network

openflow（2008）

- 提出了一种清晰的数控分离的方案



## SDN带来的机会：

#### open data plane interface（灵活性提升）

- hardware：通过标准API，更方便的操作
- software：更好的直接访问一些接口操作

#### Centralized controller

- Direct progammic control of network

#### Software abstractions on the controller 

- 解决分布式系统的问
- 方便设计net apps
- 可以设计更高级的规则，而不用去过度考虑底层的设计

## 带来的挑战

#### performance  and scalibility

#### Distribute system challenge still present

- 弹性问题（控制器只是逻辑上的集中控制，实际上不太可能）
- 并不能完全了解（或者说监测到）网络的状态
- 控制器之间的一致性问题

#### Reaching agreement on data plane protocol

-  openflow/ NFV funtions/ whitebox switch/ programmable data planes？

#### Devising the right control abstractions

- Programming Openflow：相应的指令仍然太底层了（例如，如果希望在某个节点执行负载均衡等，仍然需要从最低层的数据包字段匹配开始处理）

- 另一个问题，到底怎么样的高级抽象，才是合适的？

  

### SDN的落地用例

#### Cloud virtualization

- 多租户的虚拟网络
- 动态的迁移和部署VM

#### DC的流量工程

- 接近100%的利用率
- 避免严重的拥塞发生

#### Key characteristics of the above use case

- 用更少的硬件设施来解析部署
- Exsting solutions aren’t just inconvenient， they don’t work





