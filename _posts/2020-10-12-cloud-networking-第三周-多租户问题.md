---
layout:     post
title:      【网课】cloud Networking 第三周笔记： 多租户问题
subtitle:   多租户问题
date:       2020-10-18
author:     haoran
header-img: img/post-bg-universe.jpg
catalog: true
tags: 
    - 云网络课程
    - SDN
typora-root-url: ..
---

## 多租户问题

关键需求

## Agility（敏捷） 

即可以通过更灵活的部署获得更高的资源利用率。

- location independence addressing

  目标：租户ip可以被设定在任何地方

- performance uniform 性能的一致性

  目标：VM可以获得位置无关的throughput和服务质量

- 安全性

  目标：Micro-segmentation: isolation at tenant granularity 租户粒度上的隔离

- 网络通信服务

  目标：不仅是2层的发现，广播，多播等信息也需要考虑

### Service/tenant



- 用户在公有云中租赁空间
- 在私有云中运行服务

![image-20201018182240951](/img/cloudNetworkingClass/2020-10-12-cloud-networking-%E7%AC%AC%E4%B8%89%E5%91%A8-%E5%A4%9A%E7%A7%9F%E6%88%B7%E9%97%AE%E9%A2%98/image-20201018182240951.png)

问题场景1：跨机架的扩容可能导致用户设备的ip发生变化

![image-20201018183159355](/img/cloudNetworkingClass/2020-10-12-cloud-networking-%E7%AC%AC%E4%B8%89%E5%91%A8-%E5%A4%9A%E7%A7%9F%E6%88%B7%E9%97%AE%E9%A2%98/image-20201018183159355.png)

问题二：跨机架的扩容可能会导致集群中的某一部分吞吐/时延升高

![image-20201018183730044](/img/cloudNetworkingClass/2020-10-12-cloud-networking-%E7%AC%AC%E4%B8%89%E5%91%A8-%E5%A4%9A%E7%A7%9F%E6%88%B7%E9%97%AE%E9%A2%98/image-20201018183730044.png)

问题三：多租户共用集群时的安全性问题。

![image-20201018184128882](/img/cloudNetworkingClass/2020-10-12-cloud-networking-%E7%AC%AC%E4%B8%89%E5%91%A8-%E5%A4%9A%E7%A7%9F%E6%88%B7%E9%97%AE%E9%A2%98/image-20201018184128882.png)

问题四：不同租户的通信模式不同，可能会带来额外的影响。

目标：需要一个虚拟网络提供隔离的服务。支持更细粒度的，动态的控制。