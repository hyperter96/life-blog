---
layout: post
title: "CNI part5: Village Net 的工作原理"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-part5.jpg
categories: "CNI"
date: 2023-05-27 23:52:19
tags: ['CNI', '网络', 'go']
---

之所以选择 Village Net 作为插件的名字，是希望通过 macvlan 实现一个基于二层的网络插件。而对于一个二层网络来说，内部通讯像极了一个小村庄，通讯基本靠吼（`arp`），当然还有村通网的含义，虽然简陋，但足够好用。

## 工作原理

选择 `macvlan` 实现网络插件的原因在于，对于一个「家庭级 Kubernetes 集群」来说，节点的数目并不多，但是服务并不少，只能通过端口映射（`nodeport`）对服务进行区分，而因为所有的机器本来就在同一个交换机上，IP 相对富裕，`macvlan/ipvlan` 都是简单且好实现的方案。考虑到基于 `mac` 可以利用 `dhcp` 服务，甚至可以基于 `mac` 对 pod 的 ip 进行固定，因此便尝试使用 `macvlan` 实现网络插件。

但是 `macvlan` 在跨 net namespace 中存在不少问题，比如存在独立 net namespace 时，流量会跨过 host 的协议栈，导致了基于 `iptables/ipvs` 的 cluster ip 无法正常工作。

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/villageNet1.jpg)

当然，也正是相同原因，只是使用 `macvlan` 时，宿主机和容器的网络是不互通的，不过可以创建额外的 `macvlan bridge` 解决。

为了解决 cluster ip 无法正常工作的问题，便舍弃了只是用 `macvlan` 的念头，使用多网络接口进行组网。

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/villageNet2.jpg)

每个 Pod 都有两个网络接口，一个是基于 `bridge` 的 `eth0`，并作为默认网关，同时，在宿主机上会添加相关路由以确保可以跨节点通信。第二个接口是 `bridge` 模式的 `macvlan`，并为这个设备分配宿主机网段的 ip。

## 工作流程

和前面提到的 CNI 的工作流程一致，village net 也是分为 main 插件和 ipam 插件。

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/villageNet3.jpg)

ipam 的主要任务是基于配置从两个网段中个分配出一个可用 IP，`main` 插件是基于两个网段的 IP 创建出 `bridge、veth、macvlan` 设备，并进行配置。

## 总结

Village Net 的实现还是比较简单，甚至还需要部分手动操作，比如 `bridge` 的路由部分。但是功能上基本达到预期，而且对 cni 的坑完整的梳理了一遍。cni 本身并不复杂，但是有很多细节是在一开始做的时候没有考虑到的，甚至最后只是通过了若干 workaround 绕过。如果后面还有时间和精力放在网络插件上，再考虑如何优化。<(￣▽￣)/