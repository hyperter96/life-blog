---
layout: post
title: "服务异常故障记录：error while creating mount source path: file exists"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/abnormal-2.jpg
categories: "k8s"
date: 2023-06-06 11:45:56
tags: ['k8s']
---
前台打开页面出现以下异常，

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/abnormal-2-figure1.png)

后台k8s集群执行
```bash
kubectl describe pod atwanweb-package-d9f58ffc8-9j24b -n 464446840910139393
```

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/abnormal-2-figure2.png)