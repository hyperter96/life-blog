---
layout: post
title: "Seata part1: seata在分布式事务上的应用"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part1.jpg
categories: "分布式事务"
date: 2023-05-29 23:52:19
tags: ['seata', '分布式事务', 'go']
---

Seata 的前身是阿里巴巴集团内大规模使用保证分布式事务一致性的中间件，Seata 是其开源产品，由社区维护。在介绍 Seata 前，先与大家讨论下我们业务发展过程中经常遇到的一些问题场景。

## 业务场景

我们业务在发展的过程中，基本上都是从一个简单的应用，逐渐过渡到规模庞大、业务复杂的应用。这些复杂的场景难免遇到分布式事务管理问题，Seata 的出现正是解决这些分布式场景下的事务管理问题。介绍下其中几个经典的场景：

### 场景一: 分库分表场景下的分布式事务

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part1-figure1.jpg)

起初我们的业务规模小、轻量化，单一数据库就能保障我们的数据链路。但随着业务规模不断扩大、业务不断复杂化，通常单一数据库在容量、性能上会遭遇瓶颈。通常的解决方案是向分库、分表的架构演进。此时，即引入了分库分表场景下的分布式事务场景。

### 场景二: 跨服务场景下的分布式事务

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part1-figure2.jpg)

降低单体应用复杂度的方案：应用微服务化拆分。拆分后，我们的产品由多个功能各异的微服务组件构成，每个微服务都使用独立的数据库资源。在涉及到跨服务调用的数据一致性场景时，就引入了跨服务场景下的分布式事务。

## Seata架构

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part1-figure3.jpg)

* Transaction Coordinator（TC） 事务协调器，维护全局事务的运行状态，负责协调并驱动全局事务的提交或回滚。
* Transaction Manager（TM） 控制全局事务的边界，负责开启一个全局事务，并最终发起全局提交或全局回滚的决议，TM 定义全局事务的边界。
* Resource Manager（RM） 控制分支事务，负责分支注册、状态汇报，并接收事务协调器的指令，驱动分支（本地）事务的提交和回滚。RM 负责定义分支事务的边界和行为。

以上三个组件相互协作，TC 以 Seata 服务器（Server）形式独立部署，TM 和 RM 则是以 Seata Client 的形式集成在微服务中运行，其整体工作流程如下图。

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part1-figure4.png)

Seata 的整体工作流程如下：

* TM 向 TC 申请开启一个全局事务，全局事务创建成功后，TC 会针对这个全局事务生成一个全局唯一的 XID；
* XID 通过服务的调用链传递到其他服务;
* RM 向 TC 注册一个分支事务，并将其纳入 XID 对应全局事务的管辖；
* TM 根据 TC 收集的各个分支事务的执行结果，向 TC 发起全局事务提交或回滚决议；
* TC 调度 XID 下管辖的所有分支事务完成提交或回滚操作。
