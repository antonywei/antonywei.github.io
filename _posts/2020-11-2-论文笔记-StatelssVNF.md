---
layout:     post
title:      【论文笔记】Stateless Network Functions
subtitle:   Stateless Network Functions: Breaking the Tight Coupling of State and Processing
date:       2020-11-2
author:     haoran
header-img: img/lake.png
catalog: true
tags: 
    - NFV
    - Network stack

typora-root-url: ..

---



## Stateless Network Functions: Breaking the Tight Coupling of State and Processing

### 简介

一篇2017年的NSDI，主要提出了一种无状态的VNF，即VNF只处理逻辑，而将状态集中式存储在远程数据库进行调用，以此来实现无状态的VNF部署。

- DPDK处理数据包
- RAMCloud数据库 over RDMA存储状态
- 网络编排器对其进行控制


### Intro 

- VNF的状态使得扩展和弹性，多径传输，以及故障恢复变得难以处理。

  例如：防火墙规则、IDS的字符串匹配，NAT中的地址转换规则，负载均衡器中的有状态的防火墙。

- 现有的几种处理方法：

  - 增加冗余设备：开销成本过大
  - 不使用有状态的VNF： 例如 google maglev，但是只能局限在少部分的应用场景

  - 定期检查（监测的手段），通过登记数据包逻辑来复现状态崩溃时的场景；（可能造成计算资源开销变大的问题）
  - 通过修改VNF使其支持状态迁移（同样可能造成额外的延迟），同时存在实现上的困难(opennf方案的问题)

- 采用无状态的设计架构，即将状态分离出来进行集中式存储，处理单元通过访问集中式存储在数据库的方式，解决相应的需求

  - 快速恢复 
  - 弹性，敏捷（agility）
  - 支持多径传输
  
  ![image-20201105170450289](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201105170450289.png)

## Motivation：

### 关于状态的问题：

  - 静态的状态可以搭载在配置文件中随时启动（如，防火墙的规则等）

  - 动态的状态，需要随流量变化而变化的：

    - 和实例状态相关的（如配置等）
    - 和网络流量状态相关的：大部分操作以读为主，写入为辅（例如某一条流只有在创建的第一个包，才需要对其进行调整）

    - （原文）Finally, network functions share a common pipeline design where there is typically a lookup operation when the packet is first being processed, and sometimes a write operation after the packet has been processed

    原文内容： 

  ![image-20201105171825844](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201105171825844.png)

- 一个分离状态的例子：

  负载均衡器：

  ![image-20201105172037154](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201105172037154.png)

  对于一条流来说，只有第一个数据包需要解决flow mapping的问题，其他的只需要读即可。





## 问题的场景分析 （这部分比较有意思，影响到之后的设计）

### 处理实例故障的问题

![image-20201106163505112](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201106163505112.png)

- 发生故障后新的实例无法同步状态，但采用备份的办法会导致开销增加





### 弹性伸缩的问题

![image-20201106164623269](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201106164623269.png)

- 需要等待流完成以后或者状态迁移完毕后才能执行弹性伸缩

### 多径传输的问题

![image-20201106164853408](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201106164853408.png)

- 状态不一致带来的问题：A防火墙通过了SYN，但是ack走了B路径的实例，但是B由于没有匹配规则将ACK的流量ban了




## Architecture

![image-20201105172808515](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201105172808515.png)

三部分组成：

- Data store: 存储状态

  - RAMcloud  system （Redis 一种内存云的Key value store）
  - Extend with timer

- NF host：处理数据包

  - DPDK + SRIOV提供高速的处理服务
  - 利用容器进行封装
  - Infiniband to Data store (DPDK)

  ![image-20201105173730418](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201105173730418.png)

- Controller：调度和监控

  - Flood light改进版本：用于解决scaling和failure问题

  ![image-20201105173937046](/img/cloudNetworkingClass/2020-11-2-%E8%AE%BA%E6%96%87%E7%AC%94%E8%AE%B0-StatelssVNF/image-20201105173937046.png)

  

## 

