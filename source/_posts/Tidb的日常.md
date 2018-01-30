---
title: 'Tidb的日常'
date: 2018-01-04 23:47:01
type: "tags"
tags: [tidb]
---

> TiDB也使用了一段时间，持续记录一些日常，防止大家也踩到同样的坑~

<!--more-->

### raft sync-log

目前,公司使用的Tidb分别部署在生产环境和测试环境，生产环境的是在阿里云，而测试环境是在公司的机房。周末的时候，机房不知道为啥断过电，什么都不知道的我，周一来到公司发现测试环境的Tidb挂了（这是肯定的，因为停过电嘛~），然后就是怎么都启动不了，相当崩溃。一查日志发现：

![tidb-daily-1-1](http://oqipguzbl.bkt.clouddn.com/tidb-daily-1-1.png)

下面是raft的介绍:

![tidb-daily-1-4](http://oqipguzbl.bkt.clouddn.com/tidb-daily-1-4.png)

经过分析，得出原因应该是raft在做副本的时候出了问题。测试环境是3个kv示例，那每条数据将会有2个node去持有。node1已经将raft-log写入磁盘，而node2在写入时，发生了断电，导致node1，node2的服务和磁盘都挂掉，造成了raft日志出错。

避免这样的问题，可以设置参数（类似mysql的``innodb_flush_log_at_trx_commit``），保证每个数据的提交都能落地。不过这样就会降低性能。

![tidb-daily-1-2](http://oqipguzbl.bkt.clouddn.com/tidb-daily-1-2.jpg)

联系了Tidb的开发人员，处理该情况的工具还没有完善（该问题已有[issues](https://github.com/pingcap/tidb/issues/4731)），没办法删掉出错的数据，因为测试环境的数据量比较大，迁移出来太费时，还好数据不是很重要，清理了数据就可以启动了。当时Tidb的版本如下：

![tidb-daily-1-3](http://oqipguzbl.bkt.clouddn.com/tidb-daily-1-3.png)

所以小伙伴们在部署Tidb的时候要考量是否可能出现同时挂掉的可能（虽然概率不大），尤其是对于一些关键的业务，需要更多的考虑，比如更多的副本，或者异地机房，毕竟将``sync-log``设为true会带来相当大的性能损耗。

待续...