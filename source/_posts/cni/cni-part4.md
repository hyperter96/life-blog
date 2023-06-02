---
layout: post
title: "CNI part4: CNI插件的实现"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/cni-part4.jpg
categories: "CNI"
date: 2023-05-26 23:52:19
tags: ['CNI', '网络', 'go']
---

## CNI 注册方式

CNI Plugin 的仓库在：[https://github.com/containernetworking/plugins](https://github.com/containernetworking/plugins) 。在里面可以看到每种类型 Plugin 的具体实现。每个 Plugin 都需要实现以下三个方法，再在 `main` 中注册一下。

```go
func cmdCheck(args *skel.CmdArgs) error {
    ...
}

func cmdAdd(args *skel.CmdArgs) error {
    ...
}

func cmdDel(args *skel.CmdArgs) error {
    ...
}
```

以 `host-local` 为例，注册的方法如下，需要指明上面实现的三个方法、支持的版本、以及 Plugin 的名称。

## cnitool

社区提供了一个工具 `cnitool`，是模拟 CNI 接口被调用的工具，可以在一个已存在的 `network namespace` 中增加或删除网络设备。 先来看下 `cnitool` 的执行逻辑：

```go
func main() {
	...
	netconf, err := libcni.LoadConfList(netdir, os.Args[2])
    ...
	netns := os.Args[3]
	netns, err = filepath.Abs(netns)
    ...
	// Generate the containerid by hashing the netns path
	s := sha512.Sum512([]byte(netns))
	containerID := fmt.Sprintf("cnitool-%x", s[:10])
	cninet := libcni.NewCNIConfig(filepath.SplitList(os.Getenv(EnvCNIPath)), nil)

	rt := &libcni.RuntimeConf{
		ContainerID:    containerID,
		NetNS:          netns,
		IfName:         ifName,
		Args:           cniArgs,
		CapabilityArgs: capabilityArgs,
	}

	switch os.Args[1] {
	case CmdAdd:
		result, err := cninet.AddNetworkList(context.TODO(), netconf, rt)
		if result != nil {
			_ = result.Print()
		}
		exit(err)
	case CmdCheck:
		err := cninet.CheckNetworkList(context.TODO(), netconf, rt)
		exit(err)
	case CmdDel:
		exit(cninet.DelNetworkList(context.TODO(), netconf, rt))
	}
}
```

从上面的代码中可以看出，先是从 cni 配置文件中解析出配置 `netconf`，然后获取 `netns、containerId` 等信息作为容器的运行时信息传给接口 `cninet.AddNetworkList`。 接下来看下接口 `AddNetworkList` 的实现：

```go
// AddNetworkList executes a sequence of plugins with the ADD command
func (c *CNIConfig) AddNetworkList(ctx context.Context, list *NetworkConfigList, rt *RuntimeConf) (types.Result, error) {
	var err error
	var result types.Result
	for _, net := range list.Plugins {
		result, err = c.addNetwork(ctx, list.Name, list.CNIVersion, net, result, rt)
		if err != nil {
			return nil, err
		}
	}
    ...
	return result, nil
}
```

显然，该函数的作用就是按顺序执行各个 Plugin 的 `addNetwork` 操作。再看下 `addNetwork` 函数：

```go
func (c *CNIConfig) addNetwork(ctx context.Context, name, cniVersion string, net *NetworkConfig, prevResult types.Result, rt *RuntimeConf) (types.Result, error) {
	c.ensureExec()
	pluginPath, err := c.exec.FindInPath(net.Network.Type, c.Path)
    ...

	newConf, err := buildOneConfig(name, cniVersion, net, prevResult, rt)
    ...
	return invoke.ExecPluginWithResult(ctx, pluginPath, newConf.Bytes, c.args("ADD", rt), c.exec)
}
```

对每个插件的 addNetwork 操作分为三个部分：

* 首先，调用 `FindInPath` 函数，根据插件的类型来查找插件的绝对路径；
* 接着，调用 `buildOneConfig` 函数，从 `NetworkList` 中提取中当前插件的 `NetworkConfig` 结构，而其中的 `preResult` 是上一个插件的执行结果。
* 最后，调用 `invoke.ExecPluginWithResult` 函数，真正执行插件的 `Add` 操作。其中 `newConf.Bytes` 存放 `NetworkConfig` 结构以及上一个插件的执行结果编码形成的字节流；而 `c.args` 函数用于构建一个 `Args` 类型的实例，主要存储容器运行时信息以及执行 CNI 的操作信息。

事实上，`invoke.ExecPluginWithResult` 仅仅是一个包装函数，里面调用了一下 `exec.ExecPlugin` 就返回了，这里我们看一下 `exec.ExecPlugin` 的实现：

```go
func (e *RawExec) ExecPlugin(ctx context.Context, pluginPath string, stdinData []byte, environ []string) ([]byte, error) {
	stdout := &bytes.Buffer{}
	stderr := &bytes.Buffer{}
	c := exec.CommandContext(ctx, pluginPath)
	c.Env = environ
	c.Stdin = bytes.NewBuffer(stdinData)
	c.Stdout = stdout
	c.Stderr = stderr

	// Retry the command on "text file busy" errors
	for i := 0; i <= 5; i++ {
		err := c.Run()
        ...
		// All other errors except than the busy text file
		return nil, e.pluginErr(err, stdout.Bytes(), stderr.Bytes())
	}
    ...
}
```

看到这里，我们也就看到了整个 CNI 的核心逻辑，出乎意料的简单，仅仅是 `exec` 了插件的可执行文件，发生错误的时候重试 5 次。

至此，整个 CNI 的执行流程已经非常清晰了，简而言之，一个 CNI 插件就是一个可执行文件，从配置文件中获取网络的配置信息，从容器运行时获取容器的信息，前者以标准输入的形式，后者以环境变量的形式传给各个插件，最终以配置文件中定义的顺序依次调用各个插件，并且将前一个插件的执行结果包含在配置信息中传给下一个插件。

尽管如此，我们目前熟悉的成熟的网络插件的方案（如 `calico`），通常都不是依次调用 Plugin，而是只调用 `main` 插件，在 `main` 插件中调用 `ipam` 插件，并当场获取执行结果。