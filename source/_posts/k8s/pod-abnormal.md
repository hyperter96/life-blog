---
layout: post
title: "基于真实场景解读K8S_Pod的各种异常"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/k8s-2.jpg
categories: "k8s"
date: 2023-05-06 11:45:56
tags: ['k8s']
---

# Pod 生命周期

在整个生命周期中，Pod 会出现 5 种阶段（Phase）。

- **Pending**：Pod 被 K8s 创建出来后，起始于 Pending 阶段。在 Pending 阶段，Pod 将经过调度，被分配至目标节点开始拉取镜像、加载依赖项、创建容器。
- **Running**：当 Pod 所有容器都已被创建，且至少一个容器已经在运行中，Pod 将进入 Running 阶段。
- **Succeeded**：当 Pod 中的所有容器都执行完成后终止，并且不会再重启，Pod 将进入 Succeeded 阶段。
- **Failed**：若 Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止，也就是说容器以非 0 状态异常退出或被系统终止，Pod 将进入 Failed 阶段。
- **Unknown**：因为某些原因无法取得 Pod 状态，这种情况 Pod 将被置为 Unknown 状态。

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/k8s-pod-abnormal.png)


一般来说，对于 Job 类型的负载，Pod 在成功执行完任务之后将会以 Succeeded 状态为终态。而对于 Deployment 等负载，一般期望 Pod 能够持续提供服务，直到 Pod 因删除消失，或者因异常退出/被系统终止而进入 Failed 阶段。

容器也有其生命周期状态（State）：Waiting、Running 和 Terminated。并且也有其对应的状态原因（Reason），例如 ContainerCreating、Error、OOMKilled、CrashLoopBackOff、Completed 等。而对于发生过重启或终止的容器，上一个状态（LastState）字段不仅包含状态原因，还包含上一次退出的状态码（Exit Code）。例如容器上一次退出状态码是 137，状态原因是 OOMKilled，说明容器是因为 OOM 被系统强行终止。在异常诊断过程中，容器的退出状态是至关重要的信息。

除了必要的集群和应用监控，一般还需要通过 kubectl 命令搜集异常状态信息。

```bash
// 获取Pod当前对象描述文件
kubectl get po <podName> -n <namespace> -o yaml

// 获取Pod信息和事件（Events）
kubectl describe pod <podName> -n <namespace>

// 获取Pod容器日志
kubectl logs <podName> <containerName> -n <namespace>

// 在容器中执行命令
kubectl exec <podName> -n <namespace> -c <containerName> -- <CMD> <ARGS>
```

# Pod 异常场景

Pod 在其生命周期的许多时间点可能发生不同的异常，按照 Pod 容器是否运行为标志点，我们将异常场景大致分为两类：

1. 在 Pod 进行调度并创建容器过程中发生异常，此时 Pod 将卡在 Pending 阶段。
2. Pod 容器运行中发生异常，此时 Pod 按照具体场景处在不同阶段。

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/abnormal-scenario.jpeg)


## 1. 调度失败

{% note primary flat %}
常见错误状态：`Unschedulable`
{% endnote %}

Pod 被创建后进入调度阶段，K8s 调度器依据 Pod 声明的资源请求量和调度规则，为 Pod 挑选一个适合运行的节点。当集群节点均不满足 Pod 调度需求时，Pod 将会处于 Pending 状态。造成调度失败的典型原因如下：

* 节点资源不足

    K8s 将节点资源（CPU、内存、磁盘等）进行数值量化，定义出节点资源容量（Capacity）和节点资源可分配额（Allocatable）。资源容量是指 Kubelet 获取的计算节点当前的资源信息，而资源可分配额是 Pod 可用的资源。Pod 容器有两种资源额度概念：请求值 Request 和限制值 Limit，容器至少能获取请求值大小、至多能获取限制值的资源量。Pod 的资源请求量是 Pod 中所有容器的资源请求之和，Pod 的资源限制量是 Pod 中所有容器的资源限制之和。K8s 默认调度器按照较小的请求值作为调度依据，保障可调度节点的资源可分配额一定不小于 Pod 资源请求值。当集群没有一个节点满足 Pod 的资源请求量，则 Pod 将卡在 Pending 状态。

    Pod 因为无法满足资源需求而被 Pending，可能是因为集群资源不足，需要进行扩容，也有可能是集群碎片导致。以一个典型场景为例，用户集群有 10 几个 4c8g 的节点，整个集群资源使用率在 60%左右，每个节点都有碎片，但因为碎片太小导致扩不出来一个 2c4g 的 Pod。一般来说，小节点集群会更容易产生资源碎片，而碎片资源无法供 Pod 调度使用。如果想最大限度地减少资源浪费，使用更大的节点可能会带来更好的结果。

* 超过 Namespace 资源配额

    K8s 用户可以通过资源配额（Resource Quota）对 Namespace 进行资源使用量限制，包括两个维度：

    * 限定某个对象类型（如 Pod）可创建对象的总数
    * 限定某个对象类型可消耗的资源总数

  如果在创建或更新 Pod 时申请的资源超过了资源配额，则 Pod 将调度失败。此时需要检查 Namespace 资源配额状态，做出适当调整。

* 不满足 `NodeSelector` 节点选择器

    Pod 通过 `NodeSelector` 节点选择器指定调度到带有特定 Label 的节点，若不存在满足 NodeSelector 的可用节点，Pod 将无法被调度，需要对 `NodeSelector` 或节点 Label 进行合理调整。

* 不满足亲和性

    节点亲和性（Affinity）和反亲和性（Anti-Affinity）用于约束 Pod 调度到哪些节点，而亲和性又细分为软亲和（Preferred）和硬亲和（Required）。对于软亲和规则，K8s 调度器会尝试寻找满足对应规则的节点，如果找不到匹配的节点，调度器仍然会调度该 Pod。而当硬亲和规则不被满足时，Pod 将无法被调度，需要检查 Pod 调度规则和目标节点状态，对调度规则或节点进行合理调整。

* 节点存在污点

    K8s 提供污点（Taints）和容忍（Tolerations）机制，用于避免 Pod 被分配到不合适的节点上。假如节点上存在污点，而 Pod 没有设置相应的容忍，Pod 将不会调度到该 节点。此时需要确认节点是否有携带污点的必要，如果不必要的话可以移除污点；若 Pod 可以分配到带有污点的节点，则可以给 Pod 增加污点容忍。

* 没有可用节点

    节点可能会因为资源不足、网络不通、Kubelet 未就绪等原因导致不可用（NotReady）。当集群中没有可调度的节点，也会导致 Pod 卡在 Pending 状态。此时需要查看节点状态，排查不可用节点问题并修复，或进行集群扩容。

## 2. 镜像拉取失败

{% note primary flat %}
常见错误状态：`ImagePullBackOff`
{% endnote %}

Pod 经过调度后分配到目标节点，节点需要拉取 Pod 所需的镜像为创建容器做准备。拉取镜像阶段可能存在以下几种原因导致失败：

* 镜像名字拼写错误或配置了错误的镜像

  出现镜像拉取失败后首先要确认镜像地址是否配置错误

* 私有仓库的免密配置错误
  集群需要进行免密配置才能拉取私有镜像。自建镜像仓库时需要在集群创建免密凭证 Secret，在 Pod 指定 ImagePullSecrets，或者将 Secret 嵌入 ServicAccount，让 Pod 使用对应的 ServiceAccount。而对于 acr 等镜像服务云产品一般会提供免密插件，需要在集群中正确安装免密插件才能拉取仓库内的镜像。免密插件的异常包括：集群免密插件未安装、免密插件 Pod 异常、免密插件配置错误，需要查看相关信息进行进一步排查

* 网络不通
  网络不通的常见场景有三个：

  * 集群通过公网访问镜像仓库，而镜像仓库未配置公网的访问策略。对于自建仓库，可能是端口未开放，或是镜像服务未监听公网 IP；对于 acr 等镜像服务云产品，需要确认开启公网的访问入口，配置白名单等访问控制策略

  * 集群位于专有网络，需要为镜像服务配置专有网络的访问控制，才能建立集群节点与镜像服务之间的连接

  * 拉取海外镜像例如 `gcr.io` 仓库镜像，需配置镜像加速服务

  * 镜像拉取超时

    常见于带宽不足或镜像体积太大，导致拉取超时。可以尝试在节点上手动拉取镜像，观察传输速率和传输时间，必要时可以对集群带宽进行升配，或者适当调整 Kubelet 的 `--image-pull-progress-deadline` 和 `--runtime-request-timeout` 选项

  * 同时拉取多个镜像，触发并行度控制
    常见于用户弹性扩容出一个节点，大量待调度 Pod 被同时调度上去，导致一个节点同时有大量 Pod 启动，同时从镜像仓库拉取多个镜像。而受限于集群带宽、镜像仓库服务稳定性、容器运行时镜像拉取并行度控制等因素，镜像拉取并不支持大量并行。这种情况可以手动打断一些镜像的拉取，按照优先级让镜像分批拉取
