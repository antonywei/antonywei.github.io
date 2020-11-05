---
layout:     post
title:      【网课】cloud Networking 笔记：虚拟网络 VL2 of microsoft
subtitle:   SDN网络
date:       2020-10-27
author:     haoran
header-img: img/post-bg-universe.jpg
catalog: true
tags: 
    - 云网络课程
    - SDN
    - 虚拟网络
typora-root-url: ..
---

摘自[CLOS 架构](https://www.zhihu.com/question/48343492) （回顾一下第一节P-FAT-TREE的内容）



特点：

- 非常暴力，直接堆料
- 又多少带宽接入，接有多少带宽上联，输入输出带宽是1：1的
- 但是解决了leaf-spine种的服务器流量超额认购的问题

DC内处理请求的过程（极简版本）

1. 通过VIP（虚拟IP）接收外部请求
2. 通过三层边界路由器（BR）、和接入路由器(AR)被路由到二层域
3. 通过负载均衡器（LB）讲服务负载到对应的DIP(直接IP地址)对应的服务器上

部分摘自[VL2 的一篇介绍](https://www.sdnlab.com/20522.html) 以及 Cloud Networking class

## motivation 

- DC内部跨服务器流量已经成为瓶颈（4x于与外部的通信流量，现在可能更多）
- DC内部的流量模式是不可预测的。。

![image-20201027223715883](/img/cloudNetworkingClass/2020-10-27-cloud-networking-VL2/image-20201027223715883.png)

将100s间隔的DC内流量分类进行分类，可以发现，流量的变化情况非常快，而且对于特定的流量矩阵没有明显的模式

- 网络性能已经成为了服务器的瓶颈
- Server agent: Query server, wrap AAs in outer LA header





### VL2 架构

![image-20201027224802476](/img/cloudNetworkingClass/2020-10-27-cloud-networking-VL2/image-20201027224802476.png)

*“All problem in computer science can be solved by another level of indirection. "—— by David Wheeler*

VL2正运用了应用这种思想

三部分交换机：

- TOR (服务器交换机)
- Aggr (汇聚交换机)
- Int (中继交换机)

中继交换机和汇聚交换机组成**完全二分图**（意味着每个Aggr到其他的都有N条链路可以通信，N是Int的数量

设计思路

1. App/Tenant layer

   -  Application Addresses(AAs)：位置无关的
   - 交换机对APP而言呈现为一个巨大switch（跨网交换机传输是无感知的）
2. 虚拟层
   - Directory server: 维持AAs-LAs的映射
   - server agent Query server， wrap AAs in outer LA header.


3. 物理层
   - Locator Addresses(LAs)：和拓扑，路由挂钩
   - 网络层通过 OSPF路由





### VL2寻址方式

![img](/img/cloudNetworkingClass/2020-10-27-cloud-networking-VL2/DC-VL2-fig-6.jpg)

----

如图所示，服务器协议栈中增加shim子层，TOR交换机隧道、目录系统实现寻址。

#### 寻址方式流程

![image-20201028154404059](/img/cloudNetworkingClass/2020-10-27-cloud-networking-VL2/image-20201028154404059.png)

1. 应用层服务器S（20.0.0.55）发送ARP包请求服务器D（20.0.0.56）的地址
2. shim子层将ARP拦截（不再用arp广播），并向系统目录(DS)发送数据，请求D的LAs地址（10.0.0.6）
3. 目录系统中记录着AAs-LAs的映射关系，其中AAs为服务器地址，LAs为交换机地址，因此，目录系统收到s的请求后，返回给Shim层的是D的服务器所串联的ToR交换机地址（10.0.0.6）
4. shim层收到目录系统应答后，将数据包封装，其目的地址为D的ToR地址（10.0.0.6）。然后将数据包发送给自己的ToR交换机。ToR通过汇聚，二次封装，通过汇聚交换机、中继交换机发送的D的交换机。
5. D的ToR交换机收到数据包后，将相应的数据包解封，获取真实的目的地址（20.0.0.56）并发送给服务器。

### VL2的负载均衡和多径传输

![image-20201027225059270](/img/cloudNetworkingClass/2020-10-27-cloud-networking-VL2/image-20201027225059270.png)

- 利用VLB实现负载均衡
- ECMP实现多径传输（所有的都是3跳，所以区别不大）
- 通过设置给Int交换机相同的地址，大幅度减少路由的表项数量。

### VL2目录更新（和寻址部分内容相关）

- RSM(Replicated State Machine) Server：存储状态，并保证多个目录服务器中AAs-LAs的一致性、并用于备份，容灾等。。
- DS(Directory Server)服务器：用于读取AAs-LAs，每个DS会缓存RSM中所有的AAs-LAs的映射，每30s同步一次

![img](/img/cloudNetworkingClass/2020-10-27-cloud-networking-VL2/DC-VL2-fig-7.jpg)

#### 虚拟机的迁移

1. 虚拟机迁移前，主动向DS服务器发送更新消息。
2. DS将更新消息发送给RSM服务器
3. RSM服务器收到消息后，更新自己AAs-LAs映射关系
4. 复制这个更新到所有其他的RSM，进行备份，冗余
5. 回复DS服务器ACK，表示已经确认更新
6. 通知所有的进行映射更新

#### 被动更新机制

1. 某DS收到一条（可能是陈旧的）映射请求
2. DS会对映射仍然继续继续响应
3. 传输响应的数据包给目的ToR
4. 由于ToR发现目的服务器不在自己域内，会向DS转发信息，通知DS此映射已经过期，出发DS映射更新。

### 分析评估

#### agility

1. 位置无关的的地址
   - AAs 是位置无关的地址
2. L2 层网络的语义保持相同（兼容问题）
   - Agents解决了二层广播、多播的问题

- 通过增加一个2.5层的网络实现

#### Performance uniformity

1. Clos network is nonblocking
2. uniform capacity everywhere
3. ECMP provides goods load balance
4. **性能此时的瓶颈主要依赖于终端的网络处理能力。**
5. **给更快速的负载均衡提供了机会**

#### Security

- Directory system can allow/deny connections  by choosing whether to resolve an AA to a LA
- **子网的划分暂时没有清晰的分析。**

#### 其他的关键的思想：

1. Directory servers: 逻辑集中控制的
   - 编排应用位置
   - 控制转发逻辑
2. Host agents：提供了一个动态的，"可编程"的数据平面

### 总结

优点：

- 提供了一种寻址方式解决资源分段的问题。
- 通过clos组网的方式解决不同集群间相互通信时延，带宽差异的问题。
- 支持VLB，ECMP提高利用率和传输性能。

缺点：

- 需要修改服务器的协议栈
- 需要高性能、低延迟的DS来提供查找服务。

