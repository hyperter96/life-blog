---
layout: post
title: "Seata part2: seata的可观测性实践"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part2.jpg
categories: "分布式事务"
date: 2023-05-30 23:52:19
tags: ['seata', '分布式事务', 'go']
---

## 为什么需要可观测性

* 分布式事务消息链路较复杂Seata 在解决了用户易用性和分布式事务一致性这些问题的同时，需要多次 TC 与 TM、RM 之间的交互，尤其当微服务的链路变复杂时，Seata 的交互链路也会呈正相关性增加。这种情况下，其实我们就需要引入可观测的能力来观察、分析事物链路。
* 异常链路、故障排查难定位，性能优化无从下手在排查 Seata 的异常事务链路时，传统的方法需要看日志，这样检索起来比较麻烦。在引入可观测能力后，帮助我们直观的分析链路，快速定位问题；为优化耗时的事务链路提供依据。
* 可视化、数据可量化可视化能力可让用户对事务执行情况有直观的感受；借助可量化的数据，可帮助用户评估资源消耗、规划预算。

## 可观测性概览

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part2-figure1.jpg)

## Metrics维度

### 设计思路

* Seata 作为一个被集成的数据一致性框架，Metrics 模块将尽可能少的使用第三方依赖以降低发生冲突的风险
* Metrics 模块将竭力争取更高的度量性能和更低的资源开销，尽可能降低开启后带来的副作用
* 配置时，Metrics 是否激活、数据如何发布，取决于对应的配置；开启配置则自动启用，并默认将度量数据通过 `prometheus exporter` 的形式发布
* 不使用 Spring，使用 `SPI(Service Provider Interface)` 加载扩展


### 模块设计

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part2-figure2.jpg)

* `seata-metrics-core`：Metrics 核心模块，根据配置组织（加载）1 个 `Registry` 和 N 个 `Exporter`
* `seata-metrics-api`：定义了 `Meter` 指标接口，`Registry` 指标注册中心接口
* `seata-metrics-exporter-prometheus`：内置的 `prometheus-exporter` 实现
* `seata-metrics-registry-compact`：内置的 `Registry` 实现，并轻量级实现了 `Gauge、Counter、Summay、Timer` 指标

### Metrics 工作流

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part2-figure3.jpg)

上图是 metrics 模块的工作流，其工作流程如下：

* 利用 SPI 机制，根据配置加载 `Exporter` 和 `Registry` 的实现类
* 基于消息订阅与通知机制，监听所有全局事务的状态变更事件，并 `publish` 到 `EventBus`
* 事件订阅者消费事件，并将生成的 `metrics` 写入 `Registry`
* 监控系统（如 `prometheus`）从 `Exporter` 中拉取数据
