---
layout: post
title: "DTM part2: DTM 的 SAGA 事务模式"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part5.jpg
categories: "分布式事务"
date: 2023-06-02 23:52:19
tags: ['dtm', '分布式事务', 'go']
---

SAGA 事务模式是 DTM 中最常用的模式，主要是因为 SAGA 模式简单易用，工作量少，并且能够解决绝大部分业务的需求。

SAGA 最初出现在1987年 Hector Garcaa-Molrna & Kenneth Salem 发表的论文 SAGAS 里。其核心思想是将长事务拆分为多个短事务，由 Saga 事务协调器协调，如果每个短事务都成功提交完成，那么全局事务就正常完成，如果某个步骤失败，则根据相反顺序一次调用补偿操作。

## 拆分为子事务

例如我们要进行一个类似于银行跨行转账的业务，将A中的30元转给B，根据Saga事务的原理，我们将整个全局事务，切分为以下服务：

- 转出（TransOut）服务，这里转出将会进行操作A-30
- 转出补偿（TransOutCompensate）服务，回滚上面的转出操作，即A+30
- 转入（TransIn）服务，转入将会进行B+30
- 转入补偿（TransInCompensate）服务，回滚上面的转入操作，即B-30

整个 SAGA 事务的逻辑是：

执行转出成功=>执行转入成功=>全局事务完成

如果在中间发生错误，例如转入B发生错误，则会调用已执行分支的补偿操作，即：

执行转出成功=>执行转入失败=>执行转入补偿成功=>执行转出补偿成功=>全局事务回滚完成

下面我们看一个成功完成的 SAGA 事务典型的时序图：

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/dtm-part2-figure1.jpg)

在这个图中，我们的全局事务发起人，将整个全局事务的编排信息，包括每个步骤的正向操作和反向补偿操作定义好之后，提交给服务器，服务器就会按步骤执行前面 SAGA 的逻辑。

## SAGA 的接入

我们看看Go如何接入一个SAGA事务

```go
req := &gin.H{"amount": 30} // 微服务的请求Body
// DtmServer为DTM服务的地址
saga := dtmcli.NewSaga(DtmServer, shortuuid.New()).
  // 添加一个TransOut的子事务，正向操作为url: qsBusi+"/TransOut"， 逆向操作为url: qsBusi+"/TransOutCompensate"
  Add(qsBusi+"/TransOut", qsBusi+"/TransOutCompensate", req).
  // 添加一个TransIn的子事务，正向操作为url: qsBusi+"/TransIn"， 逆向操作为url: qsBusi+"/TransInCompensate"
  Add(qsBusi+"/TransIn", qsBusi+"/TransInCompensate", req)
// 提交saga事务，dtm会完成所有的子事务/回滚所有的子事务
err := saga.Submit()
```

上面的代码首先创建了一个 SAGA 事务，然后添加了两个子事务`TransOut、TransIn`，每个事务分支包括`action`和`compensate`两个操作，分别为`Add`函数的第一第二个参数。子事务定好之后提交给 dtm。dtm 收到 saga 提交的全局事务后，会调用所有子事务的正向操作，如果所有正向操作成功完成，那么事务成功结束。

我们前面的的例子，是基于 HTTP 协议 SDK 进行 DTM 接入，gRPC 协议的接入基本一样，详细例子代码可以在 [dtm-examples](https://github.com/dtm-labs/dtm-examples) 查看。

## 失败回滚

如果有正向操作失败，例如账户余额不足或者账户被冻结，那么 dtm 会调用各分支的补偿操作，进行回滚，最后事务成功回滚。

我们将上述的第二个分支调用，传递参数，让他失败

```go
  Add(qsBusi+"/TransIn", qsBusi+"/TransInCompensate", &TransReq{Amount: 30, TransInResult: "FAILURE"})
```

失败的时序图如下：

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/dtm-part2-figure2.jpg)

{% note primary flat %}
**补偿执行顺序**

dtm 的 SAGA 事务在1.10.0及之前，补偿操作是并发执行的，1.10.1之后，是根据用户指定的分支顺序，进行回滚的。

如果是普通 SAGA，没有打开并发选项，那么 SAGA 事务的补偿分支是完全按照正向分支的反向顺序进行补偿的。

如果是并发 SAGA，补偿分支也会并发执行，补偿分支的执行顺序与指定的正向分支顺序相反。假如并发 SAGA 指定A分支之后才能执行B，那么进行并发补偿时，DTM 保证A的补偿操作在B的补偿操作之后执行。
{% endnote %}

## 如何做补偿

当SAGA对分支A进行失败补偿时，A的正向操作可能

1. 已执行；
2. 未执行；
3. 甚至有可能处于执行中，最终执行成功或者失败是未知的。

那么对A进行补偿时，要妥善处理好这三种情况，难度很大。

dtm 提供了子事务屏障技术，自动处理上述三种情况，开发人员只需要编写好针对1的补偿操作情况即可，相关工作大幅简化，详细原理，参见下面的异常章节。

{% note primary flat %}
**失败的分支是否需要补偿**

dtm 常被问到的一个问题是，`TransIn`返回失败，那么这个时候是否还需要调用`TransIn`的补偿操作？DTM 的做法是，统一进行一次调用，这种的设计考虑点如下：

- XA, TCC 等事务模式是必须要的，SAGA 为了保持简单和统一，设计为总是调用补偿
- DTM 支持单服务多数据源，可能出现数据源1成功，数据源2失败，这种情况下，需要确保补偿被调用，数据源1的补偿被执行
- DTM 提供的子事务屏障，自动处理了补偿操作中的各种情况，用户只需要执行与正向操作完全相反的补偿即可
{% endnote %}