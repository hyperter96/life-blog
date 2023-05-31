---
layout: post
title: "CNI part3: CNI的IP地址管理插件(IPAM Plugins)"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-part3.jpg
categories: "CNI"
date: 2023-05-24 23:52:19
tags: ['CNI', '网络', 'go']
---

容器运行时在调用CNI插件创建容器网络时，CNI插件需要为容器网络接口分配和维护一个IP地址，并配置该网络接口所需要的路由，着给了CNI插件很大的灵活性，但也给它们带来了很大负担。 许多CNI插件需要重复编写相同的代码来支持用户可能需求的集中IP管理方案(如:`dhcp`, `host-local`)。为了减轻第三方CNI插件的编写负担，将IP管理功能独立出来，CNI规范定义了第二种插件类型 IP Address Management插件(IPAM插件)。这样，CNI插件就可以在执行过程中需要的时候直接调用相应的IPAM插件，由IPAM插件负责确定IP/子网、网关、路由并将这些信息返回到主CNI插件中。 IPAM插件可以从协议(例如DHCP)、本地文件系统存储的数据、网络配置文件中的ipam，或者以上这些方式的组合中来获取IP/子网、网关、路由等信息。

与CNI插件一样，IPAM插件也是通过执行可执行文件来被调用的。在预定义的路径列表中搜索可执行文件，通过`CNI_PATH`指定给CNI插件. IPAM插件必须接收所有传递给CNI插件的相同的环境变量。就像CNI插件一样，IPAM插件通过标准输入`stdin`接收CNI网络配置。

对于ADD操作，如果返回值为0，并且标准输出`stdout`中有如下的JSON格式输出，则说明IPAM插件执行成功。

```json
{
  "cniVersion": "0.4.0",
  "ips": [
      {
          "version": "<4-or-6>",
          "address": "<ip-and-prefix-in-CIDR>",
          "gateway": "<ip-address-of-the-gateway>"  // optional
      },
      ...
  ],
  "routes": [                                       // optional
      {
          "dst": "<ip-and-prefix-in-cidr>",
          "gw": "<ip-of-next-hop>"                  // optional
      },
      ...
  ]
  "dns": {                                          // optional
    "nameservers": <list-of-nameservers>            // optional
    "domain": <name-of-local-domain>                // optional
    "search": <list-of-search-domains>              // optional
    "options": <list-of-options>                    // optional
  }
}
```

注意，与常规的CNI插件不同，IPAM插件返回的是一个不包含`interfaces`字段的简化的`Result`结构，因为IPAM插件不应该关注它们的父插件配置的interfaces，除了那些有特殊要求的IPAM插件(例如dhcp IPAM插件)。

* `ips`字段是一个由插件决定的IP配置信息列表。每个列表项都是一个dictionary，描述了一个网络接口的IP配置。多个网络接口的IP配置和单个网络接口上的多个IP配置都将以`ips`列表中的不同列表项返回
  * `version`(string): “4”或“6”，对应列表项中IP地址的IP版本。提供的所有IP地址和网关必须符合给定的版本。
  * `address`(string): CIDR格式的IP地址(如“192.168.1.3/24”
  * `gateway`(string): 对应子网的默认网关(如果存在的话)。它不会要求CNI插件添加任何与该网关相关的路由。要添加的路由通过routes字段单独指定。使用该字段的一个例子是，CNI bridge插件将这个IP地址添加到Linux bridge上，将其用作网关。
  * `interface`(uint): CNI插件结果的接口列表索引，指示该IP配置应用于哪个接口。IPAM插件不应该返回这个字段，因为它们没有关于网络接口的信息。
* `routes`字段由以下内容组成。所有的IP地址在`routes`中一定要有相同的IP版本，4或者6
  * `dst`(string): 以CIDR描述的目标子网
  * `gw`(string): 网关IP。如果省略，将假定一个默认网关(由CNI插件确定)
* `dns`包含了由一些通用的DNS信息组成的dictionary
  * `nameservers`(list of strings, optional): 一个对配置网络可见的按优先级顺序排列的dns服务器列表，列表中的每个值是一个IPV4或IPV6的字符串
  * `domain`(string, optional): 用于短主机名查找的本地域名
  * `search`(list of strings, optional): 用于短主机名查找的优先级排序搜索域列表，被大多数解析器(resolver)在解析时优先于domain
  * `options` (list of strings, optional): 一组可以传递给解析器(resolver)的选项值


CNI项目目前提供了以下几个IPAM插件:

* `dhcp`: 在宿主机上运行`dhcp`守护程序，代表容器发出`dhcp`请求
* `host-local`: 维护一个分配ip的本地数据库
* `static`: 为容器分配一个静态IPv4/IPv6地址，主要用于调试