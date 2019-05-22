---
title: 'Tidb的日常'
date: 2018-01-04 23:47:01
type: "tags"
tags: [tidb]
---

> TiDB也使用了一段时间，持续记录一些日常，防止大家也踩到同样的坑~ 🥕

<!--more-->

### raft sync-log

目前,公司使用的Tidb分别部署在生产环境和测试环境，生产环境的是在阿里云，而测试环境是在公司的机房。周末的时候，机房不知道为啥断过电，什么都不知道的我，周一来到公司发现测试环境的Tidb挂了（这是肯定的，因为停过电嘛~），然后就是怎么都启动不了，相当崩溃。一查日志发现：

![tidb-daily-1-1](https://github.com/7le/7le.github.io/raw/master/image/tidb/tidb-daily-1-1.png)

下面是raft的介绍:

![tidb-daily-1-4](https://github.com/7le/7le.github.io/raw/master/image/tidb/tidb-daily-1-4.png)

经过分析，得出原因应该是raft在做副本的时候出了问题。测试环境是3个kv示例，那每条数据将会有2个node去持有。node1已经将raft-log写入磁盘，而node2在写入时，发生了断电，导致node1，node2的服务和磁盘都挂掉，造成了raft日志出错。

避免这样的问题，可以设置参数（类似mysql的``innodb_flush_log_at_trx_commit``），保证每个数据的提交都能落地。不过这样就会降低性能。

![tidb-daily-1-2](https://github.com/7le/7le.github.io/raw/master/image/tidb/tidb-daily-1-2.jpg)

联系了Tidb的开发人员，处理该情况的工具还没有完善（该问题已有[issues](https://github.com/pingcap/tidb/issues/4731)），没办法删掉出错的数据，因为测试环境的数据量比较大，迁移出来太费时，还好数据不是很重要，清理了数据就可以启动了。当时Tidb的版本如下：

![tidb-daily-1-3](https://github.com/7le/7le.github.io/raw/master/image/tidb/tidb-daily-1-3.png)

所以小伙伴们在部署Tidb的时候要考量是否可能出现同时挂掉的可能（虽然概率不大），尤其是对于一些关键的业务，需要更多的考虑，比如更多的副本，或者异地机房，毕竟将``sync-log``设为true会带来相当大的性能损耗。

### 惊魂三小时

本来今天是要升级下Tikv的配置，后来得知还需要升级下版本，然后运维喊上了我，一起弄下。

然后，就犯了几个重大的错误。记录一下，以此谨记。

#### ansible

使用ansible部署，一定要按着官方文档来，不然真的很僵硬。在生产环境，需要下载 GA 版本部署 TiDB。一开始部署现网用的是master版本，就出现了很多错误。

> 切记需要使用GA版本

```
cd /home/tidb
git clone -b release-1.0 https://github.com/pingcap/tidb-ansible.git
```
这个版本的binlog是基于kafak的，如果要local的话，有个``release-1.0-binlog-local``版本

#### 阿里云ECS设置自动挂载磁盘

因为Tikv的需要较大的磁盘，当时是直接挂载了阿里云的SSD磁盘，不过当时没有将挂载磁盘设置为开机自动挂载。导致这次升级配置的时候（阿里云ECS升级配置的时候需要重启生效），重启后，磁盘没有挂上，只升级了Tidb（Servers），Pd，而Tikv仍然是原来版本。简直吐血···

#### Thanks to Tidb Group

最后联系了Tidb团队，Tidb的大佬远程协助了我们，一堆操作猛如虎，最后升级了版本，也保证了数据。吸取教训，防止以后再跌跟头~

### Tidb 查询流程

![tidb_select](https://github.com/7le/7le.github.io/raw/master/image/tidb/tidb_select.jpg)

流程如下：

* tidb 收到来自客户端的 select 请求。
* tidb 分析该请求，准备执行计划；同时，tidb 向 pd 获取 start_ts。
* tidb 从缓存中获取 information_schema，若没有，从 tikv 获取 information_schema。
* tidb 从 information_schema 中获取到当前用户所操作的 table 的元信息。
* tidb 根据准备好的执行计划，将 tidb 这边的 keyrange 带上 table 的元信息后组织成 tikv 的 keyrange。
* tidb 从缓存或 PD 获取每个 keyrange 所在的 regions 信息。
* tidb 根据 regions 对 keyrange 进行分组。
* tidb 并发向所有 regions 对应的 tikv 分发 select 请求。
* tidb 收到所有结果后，整理数据。
* tidb 执行下一个执行计划 5，或返回客户端数据。

简单的说，就是 TiDB 在收到客户端的查询请求后，切分成一个个以 Range 为单位的子任务， 并行下发到所在的 TIKV 上。

具体的查询如：``` select count(*) from t where a+b>5 ```

![tidb_coprocessor_select](https://github.com/7le/7le.github.io/raw/master/image/tidb/tidb_coprocessor_select.jpg)

假设表 t 的数据落在三个 region 上，各个 region 所包含的主键 (handle) 范围如下：

* region1 包含了 t 的 [0,100)
* region2 包含了 t 的 [100,1000)
* region3 包含了 t 的 [1000,+~)

那么只需要在各个 region 上分别统计符合条件 a+b>5 的数据，然后在 tidb 再将这三份数据聚合即可。

待续...

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)