---
layout: post
title: "CNI part2: CNI插件是怎么工作的"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-part2.jpg
categories: "CNI"
date: 2023-05-23 23:52:19
tags: ['CNI', '网络', 'go']
---

CNI的接口并不是指HTTP，gRPC接口，CNI接口是指对可执行程序的调用（`exec`)。这些可执行程序称之为CNI插件，以K8S为例，K8S节点默认的CNI插件路径为 `/opt/cni/bin` ，在K8S节点上查看该目录，可以看到可供使用的CNI插件：

```bash
$ ls /opt/cni/bin/
bandwidth  bridge  dhcp  firewall  flannel  host-device  host-local  ipvlan  loopback  macvlan  portmap  ptp  sbr  static  tuning  vlan
```

CNI的工作过程大致如下图所示：

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-workflow.png)

CNI通过JSON格式的配置文件来描述网络配置，当需要设置容器网络时，由容器运行时负责执行CNI插件，并通过CNI插件的标准输入（stdin）来传递配置文件信息，通过标准输出（stdout）接收插件的执行结果。图中的 `libcni` 是CNI提供的一个go package，封装了一些符合CNI规范的标准操作，便于容器运行时和网络插件对接CNI标准。

## 插件入参

容器运行时通过设置**环境变量**以及从**标准输入**传入的配置文件来向插件传递参数。

### 环境变量

* `CNI_COMMAND`：定义期望的操作，可以是`ADD，DEL，CHECK`或`VERSION`。
* `CNI_CONTAINERID`：容器ID，由容器运行时管理的容器唯一标识符。
* `CNI_NETNS`：容器网络命名空间的路径。（形如 `/run/netns/[nsname]`)。
* `CNI_IFNAME`：需要被创建的网络接口名称，例如`eth0`。
* `CNI_ARGS`：运行时调用时传入的额外参数，格式为分号分隔的key-value对，例如 `FOO=BAR;ABC=123`
* `CNI_PATH`: CNI插件可执行文件的路径，例如`/opt/cni/bin`。

### 配置文件

文件示例：

```json
{
  "cniVersion": "0.4.0", // 表示希望插件遵循的CNI标准的版本。
  "name": "dbnet",  // 表示网络名称。这个名称并非指网络接口名称，是便于CNI管理的一个表示。应当在当前主机(或其他管理域)上全局唯一。
  "type": "bridge", // 插件类型
  "bridge": "cni0", // bridge插件的参数，指定网桥名称。
  "ipam": { // IP Allocation Management，管理IP地址分配。
    "type": "host-local", // ipam插件的类型。
    // ipam 定义的参数
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  }
}
```

### 公共定义部分

配置文件分为公共部分和插件定义部分。公共部分在CNI项目中使用结构体`NetworkConfig`定义：

```go
type NetworkConfig struct {
   Network *types.NetConf
   Bytes   []byte
}
...
// NetConf describes a network.
type NetConf struct {
   CNIVersion string `json:"cniVersion,omitempty"`

   Name         string          `json:"name,omitempty"`
   Type         string          `json:"type,omitempty"`
   Capabilities map[string]bool `json:"capabilities,omitempty"`
   IPAM         IPAM            `json:"ipam,omitempty"`
   DNS          DNS             `json:"dns"`

   RawPrevResult map[string]interface{} `json:"prevResult,omitempty"`
   PrevResult    Result                 `json:"-"`
}
```

* `cniVersion` 表示希望插件遵循的CNI标准的版本。
* `name` 表示网络名称。这个名称并非指网络接口名称，是便于CNI管理的一个表示。应当在当前主机(或其他管理域)上全局唯一。
* `type` 表示插件的名称，也就是插件对应的可执行文件的名称。
* `bridge` 该参数属于bridge插件的参数，指定主机网桥的名称。
* `ipam` 表示IP地址分配插件的配置，`ipam.type` 则表示`ipam`的插件类型。 更详细的信息，可以参考[官方文档](https://github.com/containernetworking/cni/blob/spec-v0.4.0/SPEC.md#network-configuration)。

### 插件定义部分

配置文件最终是传递给具体的CNI插件的，因此插件定义部分才是配置文件的“完全体”。公共部分定义只是为了方便各插件将其嵌入到自身的配置文件定义结构体中，举`bridge`插件为例：

```go
type NetConf struct {
	types.NetConf // <-- 嵌入公共部分
        // 底下的都是插件定义部分
	BrName       string `json:"bridge"`
	IsGW         bool   `json:"isGateway"`
	IsDefaultGW  bool   `json:"isDefaultGateway"`
	ForceAddress bool   `json:"forceAddress"`
	IPMasq       bool   `json:"ipMasq"`
	MTU          int    `json:"mtu"`
	HairpinMode  bool   `json:"hairpinMode"`
	PromiscMode  bool   `json:"promiscMode"`
	Vlan         int    `json:"vlan"`

	Args struct {
		Cni BridgeArgs `json:"cni,omitempty"`
	} `json:"args,omitempty"`
	RuntimeConfig struct {
		Mac string `json:"mac,omitempty"`
	} `json:"runtimeConfig,omitempty"`

	mac string
}
```

各插件的配置文件文档可参考[官方文档](https://www.cni.dev/plugins/current/)。

## 插件操作类型

CNI插件的操作类型只有四种：`ADD`、`DEL`、`CHECK` 和 `VERSION`。 插件调用者通过环境变量 `CNI_COMMAND` 来指定需要执行的操作。

### ADD

`ADD` 操作负责将容器添加到网络，或对现有的网络设置做更改。具体地说，`ADD` 操作要么：

* 为容器所在的网络命名空间创建一个网络接口，或者
* 修改容器所在网络命名空间中的指定网络接口
例如通过 `ADD` 将容器网络接口接入到主机的网桥中。

{% note primary flat %}
其中网络接口名称由 `CNI_IFNAME` 指定，网络命名空间由 `CNI_NETNS` 指定。
{% endnote %}

### DEL

`DEL` 操作负责从网络中删除容器，或取消对应的修改，可以理解为是 `ADD` 的逆操作。具体地说，`DEL` 操作要么：

* 为容器所在的网络命名空间删除一个网络接口，或者
* 撤销 `ADD` 操作的修改
例如通过 DEL 将容器网络接口从主机网桥中删除。

{% note primary flat %}
其中网络接口名称由 `CNI_IFNAME` 指定，网络命名空间由 `CNI_NETNS` 指定。
{% endnote %}

### CHECK

`CHECK` 操作是`v0.4.0`加入的类型，用于检查网络设置是否符合预期。容器运行时可以通过`CHECK`来检查网络设置是否出现错误，当CHECK返回错误时（返回了一个非0状态码），容器运行时可以选择`Kill`掉容器，通过重新启动来重新获得一个正确的网络配置。

### VERSION

`VERSION` 操作用于查看插件支持的版本信息。

```bash
$ CNI_COMMAND=VERSION /opt/cni/bin/bridge
{"cniVersion":"0.4.0","supportedVersions":["0.1.0","0.2.0","0.3.0","0.3.1","0.4.0"]}
```

## 链式调用

单个CNI插件的职责是单一的，比如`bridge`插件负责网桥的相关配置， `firewall`插件负责防火墙相关配置， `portmap` 插件负责端口映射相关配置。因此，当网络设置比较复杂时，通常需要调用多个插件来完成。CNI支持插件的链式调用，可以将多个插件组合起来，按顺序调用。例如先调用 `bridge` 插件设置容器IP，将容器网卡与主机网桥连通，再调用`portmap`插件做容器端口映射。容器运行时可以通过在配置文件设置`plugins`数组达到链式调用的目的：

```json
{
  "cniVersion": "0.4.0",
  "name": "dbnet",
  "plugins": [
    {
      "type": "bridge",
      // type (plugin) specific
      "bridge": "cni0"
      },
      "ipam": {
        "type": "host-local",
        // ipam specific
        "subnet": "10.1.0.0/16",
        "gateway": "10.1.0.1"
      }
    },
    {
      "type": "tuning",
      "sysctl": {
        "net.core.somaxconn": "500"
      }
    }
  ]
}
```

`plugins`这个字段并没有出现在上文描述的配置文件结构体中。的确，CNI使用了另一个结构体——`NetworkConfigList`来保存链式调用的配置：

```go
type NetworkConfigList struct {
   Name         string
   CNIVersion   string
   DisableCheck bool
   Plugins      []*NetworkConfig 
   Bytes        []byte
}
```

但CNI插件是不认识这个配置类型的。实际上，在调用CNI插件时，需要将`NetworkConfigList`转换成对应插件的配置文件格式，再通过标准输入（`stdin`）传递给CNI插件。例如在上面的示例中，实际上会先使用下面的配置文件调用 `bridge` 插件：

```json
{
  "cniVersion": "0.4.0",
  "name": "dbnet",
  "type": "bridge",
  "bridge": "cni0",
  "ipam": {
    "type": "host-local",
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  }
}
```

再使用下面的配置文件调用`tuning`插件：

```json
{
  "cniVersion": "0.4.0",
  "name": "dbnet",
  "type": "tuning",
  "sysctl": {
    "net.core.somaxconn": "500"
  },
  "prevResult": { // 调用bridge插件的返回结果
     ...
  }
}
```

需要注意的是，当插件进行链式调用的时候，不仅需要对`NetworkConfigList`做格式转换，而且需要将前一次插件的返回结果添加到配置文件中（通过`prevResult`字段），不得不说是一项繁琐而重复的工作。不过幸好`libcni`已经为我们封装好了，容器运行时不需要关心如何转换配置文件，如何填入上一次插件的返回结果，只需要调用 `libcni`的相关方法即可。



