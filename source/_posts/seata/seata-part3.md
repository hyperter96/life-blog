---
layout: post
title: "Seata part3: seata 的 AT模式"
cover: https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part3.jpg
categories: "分布式事务"
date: 2023-05-31 23:52:19
tags: ['seata', '分布式事务', 'go']
---

Seata 提供了 AT、TCC、SAGA 和 XA 四种事务模式，可以快速有效地对分布式事务进行控制。

在这四种事务模式中使用最多，最方便的就是 AT 模式。与其他事务模式相比，AT 模式可以应对大多数的业务场景，且基本可以做到无业务入侵，开发人员能够有更多的精力关注于业务逻辑开发。

## AT模式的前提

任何应用想要使用 Seata 的 AT 模式对分布式事务进行控制，必须满足以下 2 个前提：

* 必须使用支持本地 ACID 事务特性的关系型数据库，例如 MySQL、Oracle 等；
* 应用程序必须是使用 JDBC 对数据库进行访问的 JAVA 应用。

此外，我们还需要针对业务中涉及的各个数据库表，分别创建一个 `UNDO_LOG`（回滚日志）表。不同数据库在创建 `UNDO_LOG` 表时会略有不同，以 MySQL 为例，其 `UNDO_LOG` 表的创表语句如下：

```mysql
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

## AT模式的工作机制

Seata 的 AT 模式工作时大致可以分为以两个阶段，下面我们就结合一个实例来对 AT 模式的工作机制进行介绍。

假设某数据库中存在一张名为 webset 的表，表结构如下。

| 列表   | 类型             | 主键  |
|------|----------------|-----|
| id   | `bigint(20)`   | yes |
| name | `varchar(255)` |     |
| url  | `varchar(255)`  |     |

在某次分支事务中，我们需要在 webset 表中执行以下操作。

```mysql
update webset set url = 'c.biancheng.net' where name = 'C语言中文网';
```

### 一阶段

Seata AT 模式一阶段的工作流程如下图所示。

![](https://cdn.jsdelivr.net/gh/hyperter96/hyperter96.github.io/img/seata-part3-figure1.png)

Seata AT 模式一阶段工作流程如下。

1. 获取 SQL 的基本信息：Seata 拦截并解析业务 SQL，得到 SQL 的操作类型（UPDATE）、表名（webset）、判断条件（where name = 'C语言中文网'）等相关信息。

2. 查询前镜像：根据得到的业务 SQL 信息，生成“前镜像查询语句”。

```mysql
select id,name,url from webset where  name='C语言中文网';
```

执行“前镜像查询语句”，得到即将执行操作的数据，并将其保存为“前镜像数据（beforeImage）”。

| id  | name   | url |
|-----|--------|-----|
| 1   | C语言中文网 | biancheng.net |

3. 执行业务 SQL（update webset set url = 'c.biancheng.net' where name = 'C语言中文网';），将这条记录的 url 修改为 c.biancheng.net。

4. 查询后镜像：根据“前镜像数据”的主键（id : 1），生成“后镜像查询语句”。

```mysql
select id,name,url from webset where  id= 1;
```

执行“后镜像查询语句”，得到执行业务操作后的数据，并将其保存为“后镜像数据（afterImage）”。

| id  | name   | url             |
|-----|--------|-----------------|
| 1   | C语言中文网 | c.biancheng.net |

5. 插入回滚日志：将前后镜像数据和业务 SQL 的信息组成一条回滚日志记录，插入到 UNDO_LOG 表中，示例回滚日志如下。

```json
{
  "@class": "io.seata.rm.datasource.undo.BranchUndoLog",
  "xid": "172.26.54.1:8091:5962967415319516023",
  "branchId": 5962967415319516027,
  "sqlUndoLogs": [
    "java.util.ArrayList",
    [
      {
        "@class": "io.seata.rm.datasource.undo.SQLUndoLog",
        "sqlType": "UPDATE",
        "tableName": "webset",
        "beforeImage": {
          "@class": "io.seata.rm.datasource.sql.struct.TableRecords",
          "tableName": "webset",
          "rows": [
            "java.util.ArrayList",
            [
              {
                "@class": "io.seata.rm.datasource.sql.struct.Row",
                "fields": [
                  "java.util.ArrayList",
                  [
                    {
                      "@class": "io.seata.rm.datasource.sql.struct.Field",
                      "name": "id",
                      "keyType": "PRIMARY_KEY",
                      "type": -5,
                      "value": [
                        "java.lang.Long",
                        1
                      ]
                    },
                    {
                      "@class": "io.seata.rm.datasource.sql.struct.Field",
                      "name": "url",
                      "keyType": "NULL",
                      "type": 12,
                      "value": "biancheng.net"
                    }
                  ]
                ]
              }
            ]
          ]
        },
        "afterImage": {
          "@class": "io.seata.rm.datasource.sql.struct.TableRecords",
          "tableName": "webset",
          "rows": [
            "java.util.ArrayList",
            [
              {
                "@class": "io.seata.rm.datasource.sql.struct.Row",
                "fields": [
                  "java.util.ArrayList",
                  [
                    {
                      "@class": "io.seata.rm.datasource.sql.struct.Field",
                      "name": "id",
                      "keyType": "PRIMARY_KEY",
                      "type": -5,
                      "value": [
                        "java.lang.Long",
                        1
                      ]
                    },
                    {
                      "@class": "io.seata.rm.datasource.sql.struct.Field",
                      "name": "url",
                      "keyType": "NULL",
                      "type": 12,
                      "value": "c.biancheng.net"
                    }
                  ]
                ]
              }
            ]
          ]
        }
      }
    ]
  ]
}
```

6. 注册分支事务，生成行锁：在这次业务操作的本地事务提交前，RM 会向 TC 注册分支事务，并针对主键 id 为 1 的记录生成行锁。

    {% note primary flat %}
    以上所有操作均在同一个数据库事务内完成，可以保证一阶段的操作的原子性。
    {% endnote %}

7. 本地事务提交：将业务数据的更新和前面生成的 UNDO_LOG 一并提交。

8. 上报执行结果：将本地事务提交的结果上报给 TC。

### 二阶段：提交

当所有的 RM 都将自己分支事务的提交结果上报给 TC 后，TM 根据 TC 收集的各个分支事务的执行结果，来决定向 TC 发起全局事务的提交或回滚。

若所有分支事务都执行成功，TM 向 TC 发起全局事务的提交，并批量删除各个 RM 保存的 `UNDO_LOG` 记录和行锁；否则全局事务回滚。

### 二阶段：回滚

若全局事务中的任何一个分支事务失败，则 TM 向 TC 发起全局事务的回滚，并开启一个本地事务，执行如下操作。

1. 查找 `UNDO_LOG` 记录：通过 XID 和分支事务 ID（Branch ID） 查找所有的 `UNDO_LOG` 记录。

2. 数据校验：将 `UNDO_LOG` 中的后镜像数据（afterImage）与当前数据进行比较，如果有不同，则说明数据被当前全局事务之外的动作所修改，需要人工对这些数据进行处理。

3. 生成回滚语句：根据 `UNDO_LOG` 中的前镜像（beforeImage）和业务 SQL 的相关信息生成回滚语句：

```mysql
update webset set url= 'biancheng.net' where id = 1;
```

4. 还原数据：执行回滚语句，并将前镜像数据、后镜像数据以及行锁删除。

5. 提交事务：提交本地事务，并把本地事务的执行结果（即分支事务回滚的结果）上报给 TC。