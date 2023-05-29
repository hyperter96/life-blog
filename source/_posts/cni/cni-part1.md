---
layout: post
title: "CNI part1: CNI插件在k8s集群网络充当什么角色"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-part1.jpg
categories: "CNI"
date: 2023-05-21 23:52:19
tags: ['CNI', '网络', 'go']
---

最近因为工作需要，关于K8s集群网络的知识点还是需要自己尽快上手的。于是我刚开始在想这个问题：

{% note primary flat %}
CNI插件在k8s集群网络到底充当着什么角色呢？
{% endnote %}

带着这个疑惑，我开始学习关于k8s集群网络的各种插件，譬如有: `Flannel`、 `Calico` 以及 `Kube-ovn` 等等，并且
有了一些自己的体会。现实生活中，容器网络也拥有以下常见的场景：

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-app-scenario.png)

这张图总结了七个主要使用的场景，应该也涵盖大部分运维人员网络需求。

* 固定 IP：对于现存虚拟化 / 裸机业务 / 单体应用迁移到容器环境后，都是通过 IP 而非域名进行服务间调用，此时就需要 CNI 插件有固定 IP 的功能，包括 Pod/Deployment/Statefulset。
* 网络隔离：不同租户或不同应用之间，容器组应该是不能互相调用或通信的。
* 多集群网络互联： 对于不同的 Kubernetes 集群之间的微服务进行互相调用的场景，需要多集群网络互联。这种场景一般分为 IP 可达和 Service 互通，满足不同的微服务实例互相调用需求。
* 出向限制：对于容器集群外的数据库 / 中间件，需能控制特定属性的容器应用才可访问，拒绝其他连接请求。
* 入向限制：限制集群外应用对特定容器应用的访问。
* 带宽限制：容器应用之间的网络访问加以带宽限制。
* 出口网关访问：对于访问集群外特定应用的容器，设置出口网关对其进行 SNAT 以达到统一出口访问的审计和安全需求。 理完需求和应用场景，我们来看看如何通过不同的 CNI 插件解决以上痛点。

## 术语

在对 CNI 插件们进行比较之前，我们可以先对网络中会见到的相关术语做一个整体的了解。不论是阅读本文，还是今后接触到其他和 CNI 有关的内容，了解一些常见术语总是非常有用的。



一些最常见的术语包括：

* 第 2 层网络 ： OSI（Open Systems Interconnections，开放系统互连）网络模型的“数据链路”层。第 2 层网络会处理网络上两个相邻节点之间的帧传递。第 2 层网络的一个值得注意的示例是以太网，其中 MAC 表示为子层。

* 第 3 层网络 ： OSI 网络模型的“网络”层。第 3 层网络的主要关注点，是在第 2 层连接之上的主机之间路由数据包。IPv4、IPv6 和 ICMP 是第 3 层网络协议的示例。

* VXLAN ：代表“虚拟可扩展 LAN”。首先，VXLAN 用于通过在 UDP 数据报中封装第 2 层以太网帧来帮助实现大型云部署。VXLAN 虚拟化与 VLAN 类似，但提供更大的灵活性和功能（VLAN 仅限于 4096 个网络 ID）。VXLAN 是一种封装和覆盖协议，可在现有网络上运行。

* Overlay 网络 ：Overlay 网络是建立在现有网络之上的虚拟逻辑网络。Overlay 网络通常用于在现有网络之上提供有用的抽象，并分离和保护不同的逻辑网络。

* 封装 ：封装是指在附加层中封装网络数据包以提供其他上下文和信息的过程。在 overlay 网络中，封装被用于从虚拟网络转换到底层地址空间，从而能路由到不同的位置（数据包可以被解封装，并继续到其目的地）。

* 网状网络 ：网状网络（Mesh network）是指每个节点连接到许多其他节点以协作路由、并实现更大连接的网络。网状网络允许通过多个路径进行路由，从而提供更可靠的网络。网状网格的缺点是每个附加节点都会增加大量开销。

* BGP ：代表“边界网关协议”，用于管理边缘路由器之间数据包的路由方式。BGP 通过考虑可用路径，路由规则和特定网络策略，帮助弄清楚如何将数据包从一个网络发送到另一个网络。BGP 有时被用作 CNI 插件中的路由机制，而不是封装的覆盖网络。

了解了技术术语和支持各类插件的各种技术之后，下面我们可以开始探索一些最流行的 CNI 插件了。

## Flannel

flannel是kubernetes默认提供网络插件。 Flannel 是由CoreOs团队开发社交的网络工具，CoreOS团队采用L3 Overlay模式设计flannel， 规定宿主机下各个Pod属于同一个子网，不同宿主机下的Pod属于不同的子网。

**flannel会在每一个宿主机上运行名为 `flanneld` 代理，其负责为宿主机预先分配一个子网，并为Pod分配IP地址。Flannel使用Kubernetes或etcd来存储网络配置、分配的子网和主机公共IP等信息。数据包则通过`VXLAN、UDP`或`host-gw`这些类型的后端机制进行转发。

Flannel致力于给k8s集群中的nodes提供一个3层网络，他并不控制node中的容器是如何进行组网的，仅仅关心流量如何在node之间流转。

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-flannel.png)

如上图ip为10.1.15.2的pod1与另外一个Node上的10.1.20.3的pod2进行通信。

* 首先pod1通过veth对把数据包发送到docker0虚拟网桥，网桥通过查找转发表发现10.1.20.3不在自己管理的网段，就会把数据包
转发给默认路由（这里为flannel0网桥）

* flannel.0网桥是一个vxlan设备，flannel.0收到数据包后，由于自己不是目的地10.1.20.3，也要尝试将数据包重新发送出去。数据包沿着网络协议栈向下流动，在二层时需要封二层以太包，填写目的mac地址，这时一般应该发出arp：”who is 10.1.20.3″。但vxlan设备的特殊性就在于它并没有真正在二层发出这个arp包，而是由linux kernel引发一个”L3 MISS”事件并将arp请求发到用户空间的flanned程序。

* flanned程序收到”L3 MISS”内核事件以及arp请求(who is 10.1.20.3)后，并不会向外网发送arp request，而是尝试从etcd查找该地址匹配的子网的vtep信息，也就是会找到node2上的flanel.0的mac地址信息，flanned将查询到的信息放入node1 host的arp cache表中，flanneel0完成这项工作后，linux kernel就可以在arp table中找到 10.1.20.3对应的mac地址并封装二层以太包了:

* 由于是Vlanx设备，flannel0还会对上面的包进行二次封装，封装新的以太网mac帧:

* node上2的eth0接收到上述vxlan包，kernel将识别出这是一个vxlan包，于是拆包后将packet转给node上2的flannel.0。flannel.0再将这个数据包转到docker0，继而由docker0传输到Pod2的某个容器里。

如上图，总的来说就是建立 VXLAN 隧道，通过UDP把IP封装一层直接送到对应的节点，实现了一个大的 VLAN。

## Calico

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-calico-bgp.png)

对于 Calico 这种原生对 BGP 支持很好的 CNI 插件来说，很容易实现这一点，只要两个集群通过 BGP 建立邻居，将各自的路由宣告给对方即可实现动态路由的建立。若存在多个集群，使用 BGP RR 的形式也很好解决。但这种解决方式可能不是最理想的，因为需要和物理网络环境进行配合和联调，这就需要网络人员和容器运维人员一同进行多集群网络的建设，在后期运维和管理上都有不大方便和敏捷的感觉。

## Kube-ovn

## CNI是怎么工作的