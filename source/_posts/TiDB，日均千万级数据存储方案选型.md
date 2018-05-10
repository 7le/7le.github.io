---
title: 'TiDB，日均千万级数据存储方案选型'
date: 2017-11-08 23:05:50
type: "tags"
tags: [tidb]
---

> SQL -> NoSQL -> NewSql

<!--more-->

原先的统计系统是基于mongodb存储和计算的，在考虑到之后接入新的广告系统，数据量预估会达到日均千万级别，以及更多纬度的统计和分析，和后续更多系统的接入，而且最好需要兼顾OLAP+OLTP，故此调整储存方案已经势在必行。

### 方案对比
      
#### MaxCompute（原opds）

一开始是想要依托阿里的服务，这样可以减少大量的运维成本，阿里云上对此的描述为：大数据计算服务(MaxCompute,原名ODPS)是一种快速、完全托管的TB/PB级数据仓库解决方案。
经过测试发现，此产品适用于离线分析，对于做实时的操作速度是不能接受的。操作100W的数据，做简单的聚合操作，需要30S。

![maxcompute](http://oqipguzbl.bkt.clouddn.com/tidb/maxcompute.png)

且对于用户来说没有索引的概念，用户不能自己选择创建索引，都由阿里云自己实现。提工单的时候回馈，可以得到对此产品的定位。

![工单](http://oqipguzbl.bkt.clouddn.com/tidb/%E5%B7%A5%E5%8D%95.png)

很明显MaxCompute 能实现OLAP的需求，但是确不能满足OLTP的需求。

#### ADS  （分析型数据库）

![ADS](http://oqipguzbl.bkt.clouddn.com/tidbads.png)

同样属于阿里云的服务，官方自己的评价，毫秒级别的查询速度，具体没有测试，因为考虑到成本过高，就放弃了。
          
#### TIDB （NewSql 分布式数据库）：

> 主要优势:  

* 1  支持OLAP + OLTP 满足统计系统的大方向要求，可以实现联机事务处理和联机分析处理。
* 2  高可用:
![TiDB架构](http://oqipguzbl.bkt.clouddn.com/tidb/tidb%E6%9E%B6%E6%9E%84.png)
tidb的分布式的架构，且根据PD 之间的调度和tikv之间raft算法，实现容灾。
具体文章：[技术内幕 - 谈调度](https://pingcap.com/tidb/blog-tidb-internal-3-zh)  [技术内幕 - 说存储](https://pingcap.com/tikv/blog-tidb-internal-1-zh)    
* 3  对Mysql的兼容：基本语法跟mysql无差别，几乎无缝衔接，学习成本可以大大减少。
* 4  支持水平扩容缩容：tidb支持自动扩容缩容，这个自动到底有多重要，用过mycat的就知道了。业务层不需要再去关心数据库的容量，不用去考虑分库分表，，扩容只需简单加机器就好，存储节点故障对业务透明，而且数据库本身具有自我修复的能力，保证数据不会丢失。
* 5  TIDB完美兼容mysql的binlog
* 6  目前已经可以集成spark，做复杂分析计算，后续还可以与Kubernetes整合，云服务也是大势的趋向。

> 一些不足：

* 1 目前刚刚GA，文档除了官方的之外毕竟匮乏，容易踩坑，不过github社区活跃，可以去提issues。
* 2 对于服务器要求相对较高，磁盘需要SSD才能保证性能的卓越。
                              
官网文档中mysql 和 tidb 的对比图。
![TiDB架构](http://oqipguzbl.bkt.clouddn.com/tidb/tidb%E6%9E%B6%E6%9E%84.png)
   
     
### 集群部署 — ansible方式部署

具体可以看[官方部署文档](https://pingcap.com/docs-cn/op-guide/ansible-deployment/) 
按照流程便可以完成, 文档中有服务器要求， 需要注意的是 保证机器``entos7.3``以上， 且``文件系统推荐 ext4``，且服务器之间要配置``ssh免密登录``。当前是部署的最新版本是ga版本的1.0.1，ansile支持滚动升级，详见官方文档。

如果在执行 ``ansible-playbook bootstrap.yml`` 命令的时候出现校准机器配置之类的error，
例如磁盘告警 ``dd: the write speed of tikv deploy_dir disk is too slow: MB/s < 15 MB/s``
可以修改 ``inventory.ini`` 中的参数为``false``。
![inventory.ini](http://oqipguzbl.bkt.clouddn.com/tidb/inventory.png)

如果需要配置数据data目录，可以在如下文件中配置
![data目录](http://oqipguzbl.bkt.clouddn.com/tidb/data.png)

### TiDB 测评

> TIDB 集群情况：
TIDB+PD    同一台服务器  4核16G  disk:SSD
TIKV*3     3台           4核16G  disk:SSD 

在亿级别数据量的单表（100多G）下一些简单测评：
group by + 简单聚合查询：
          ![简单聚合查询](http://oqipguzbl.bkt.clouddn.com/tidb/%E4%BA%BF%E7%BA%A7%E5%88%AB%E6%B5%8B%E8%AF%95.png)
添加字段：
          ![添加字段](http://oqipguzbl.bkt.clouddn.com/tidb/%E6%B7%BB%E5%8A%A0%E5%AD%97%E6%AE%B5.png)
删除字段：
          ![删除字段](http://oqipguzbl.bkt.clouddn.com/tidb/%E5%88%A0%E9%99%A4%E5%AD%97%E6%AE%B5.png)
删除索引：
          ![删除索引](http://oqipguzbl.bkt.clouddn.com/tidb/%E5%88%A0%E9%99%A4%E7%B4%A2%E5%BC%95.png)

添加索引: 比较慢，亿级别单边需要大概1.5小时左右。

注意：TiDB 为了保证DDL能实现Online
DDL（在执行的时候不会锁表，仍能执行DML），DDL的任务为串行，当时遇到的一个问题:[github-issues](https://github.com/pingcap/tidb/issues/4986) 一些问题可以直接去提issues。
           
性能监控tidb提供了garafan， 路径为Your_IP:3000   帐号 admin  密码 admin。
         ![storage](http://oqipguzbl.bkt.clouddn.com/tidb/storage.png)
CPU的监控：
         ![CPU](http://oqipguzbl.bkt.clouddn.com/tidb/cpu.png)
磁盘的一些参数：经测试tidb性能跟磁盘读写能力关系很大，一开始比较差的服务器中，磁盘IO成为主要瓶颈。
         ![disk](http://oqipguzbl.bkt.clouddn.com/tidb/disk.png)
         

使用sysbeanch压测的结果:  prepare 1000W的数据量进行测试。
```
time sysbench oltp.lua --db-driver=mysql --oltp-table-size=10000000 --mysql-host=127.0.0.1 --mysql-port=4000 --mysql-user=root --mysql-password=58945894 --mysql-db=test prepare
```
![prepare](http://oqipguzbl.bkt.clouddn.com/tidb/prepare.png)

查看数据库
![查看数据库](http://oqipguzbl.bkt.clouddn.com/tidb/%E6%95%B0%E6%8D%AE%E5%BA%93%E6%95%B0%E6%8D%AE.png)
          
         
测试select性能：
```
time sysbench select.lua --db-driver=mysql --oltp-table-size=10000000 --mysql-host=127.0.0.1 --mysql-port=4000 --mysql-user=root --mysql-password=58945894 --mysql-db=test --events=10000000 --report-interval=5 --threads=16 --time=60 run
```
![select](http://oqipguzbl.bkt.clouddn.com/tidb/select.png)
> 16线程，select QPS 1.7万  

测试insert性能：
```
time sysbench insert.lua --db-driver=mysql --oltp-table-size=10000000 --mysql-host=127.0.0.1 --mysql-port=4000 --mysql-user=root --mysql-password=58945894 --mysql-db=test --events=10000000 --report-interval=5 --threads=16 --time=60 run
```
![insert](http://oqipguzbl.bkt.clouddn.com/tidb/insert.png)
> 16线程，insert QPS 0.5万    

测试update性能:
```
time sysbench  update_index.lua --db-driver=mysql --oltp-table-size=10000000 --mysql-host=127.0.0.1 --mysql-port=4000 --mysql-user=root --mysql-password=58945894 --mysql-db=test--events=10000000 --report-interval=5 --threads=16 --time=60 run
```
![update](http://oqipguzbl.bkt.clouddn.com/tidb/insert.png)
> 16线程，update QPS 0.4万   

测试混合性能:
```
time sysbench oltp.lua --db-driver=mysql --oltp-table-size=10000000  --mysql-host=127.0.0.1 --mysql-port=4000 --mysql-user=root --mysql-password=58945894  --mysql-db=test --events=10000000 --report-interval=5 --threads=16 --time=60 run
```
![update](http://oqipguzbl.bkt.clouddn.com/tidb/%E6%B7%B7%E5%90%88%E8%AF%BB%E5%86%99.png)
> 16线程，混合读写 QPS 0.8万

其他人不同的配置下的[测评](https://zhuanlan.zhihu.com/p/30572262)

最后选择的方案是TiDB，关于TiDB后续还有什么值得分享还会继续更新。本人才疏学浅，希望大家可以多多指教~

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)

       
