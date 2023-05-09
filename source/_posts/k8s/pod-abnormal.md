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

## 1. 调度失败

{% note danger flat %}
primary 提示块标签
{% endnote %}
