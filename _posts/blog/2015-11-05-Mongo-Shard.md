---
layout: post
title: OpenStack Ceilometer -- 后台数据存储优化之MongoDB的分片存储设置
description: OpenStack,Ceilometer,MongoDB,Shard
category: blog
---

##文档

* [MongoDB官方手册](https://docs.mongodb.org/manual/)
* [MongoDB Sharding官方手册](https://docs.mongodb.org/manual/core/sharding-introduction/)
* [SQL to Aggregation Mapping Chart官方手册](https://docs.mongodb.org/manual/reference/sql-aggregation-comparison/)
* [Expire Data from Collections by Setting TTL官方手册](https://docs.mongodb.org/manual/tutorial/expire-data/)
* [Using a sharding/replicaSet mongodb with ceilometer](https://blog.sileht.net/using-a-shardingreplicaset-mongodb-with-ceilometer.html)
* [Ceilometer Database data TTL](https://blueprints.launchpad.net/ceilometer/+spec/db-ttl)
* [Ceilometer Database data TTL Review](https://review.openstack.org/#/c/30635/)
* [MongoDB实战（11）Sharding分片（上）](http://janephp.blog.51cto.com/4439680/1330656/)
* [MongoDB实战（11）Sharding分片（下）](http://janephp.blog.51cto.com/4439680/1331401/)
* [MongoDB实战（12）Replica Sets + Sharding](http://janephp.blog.51cto.com/4439680/1332140/)
* [Mongodb千万级数据在python下的综合压力测试及应用探讨](http://cloud.51cto.com/art/201311/418290.htm)
* [配置MongoDB集群分片](http://my.oschina.net/zhzhenqin/blog/97268?p=4#comments)
* [MongoDB 自动分片 auto sharding](http://blog.fens.me/mongodb-shard/)这篇文章有点儿怪，它只是在介绍分片，并不是自动分片呀，而且MongoDB提供自动分片功能吗？

##关于MongoDB

MongoDB中的概念与关系型数据库之间的对应：

* Database --> Database
* Collection --> Table
* Document --> Row

MongoDB相较于关系型数据库的优势：

* 简化关系型数据库复杂的关联问题
* 摆脱关系模型里面的强一致性限制
* MongoDB可以做到水平扩展和高可用

学习MongoDB有几个比较重要的方面：

* CRUD操作
* 聚合(Aggregation)操作
* 索引(Indexs)
* 存储引擎(Storage)
* 复制集(Replication)
* 分片(Sharding)
* 各种命令

## MongoDB的部署

* ```yum info mongo-10gen```查看yum源中是否包含MongoDB的相关资源。

* ```vi /etc/yum.repos.d/10gen.repo```添加yum源，若已有则不添加。

    ```
    [10gen]
    name=10gen Repository
    baseurl=http://downloads-distro.mongodb.org/repo/redhat/os/x86_64
    gpgcheck=0
    ```

* ```yum info mongo-10gen-server```，配置好yum源之后，查看yum源中是否包含MongoDB的服务器包的信息。

* 安装MongoDB的服务器端和客户端工具：

    ```
    yum install mongo-10gen-server
    yum install mongo-10gen
    ```

* 根据需要修改```/etc/mongod.conf```，启动MongoDB: ```service mongod start```。

## MongoDB的简单操作

###连接MongoDB

* 相关操作如下：

    ```
    [root@node-51 ~]# mongo --host hostIP/hostName --port portNum
    mongos> show dbs
    admin   *GB
    ceilometer  *GB
    config  *GB
    mongos> use ceilometer
    mongos> show collections
    meter
    project
    resource
    system.indexes
    system.users
    user
    ```

###查询meter中所有的数据

* 相关操作如下：

    ```
    mongos> db.meter.find()
    mongos> db.meter.find().count()
    ```

###查询meter中所有的counter_name

* 相关操作如下：

    ```
    mongos> db.meter.distinct("counter_name")
    ```

###查询meter中各counter_name有多少条记录

* 相关操作如下：

    ```
    mongos> db.meter.aggregate([
        {
            $group: {
                _id: "$counter_name",
                count: {$sum:1}
            }
        },
        { $match: { count: { $gt: 1 } } }
    ])
    ```

我们一般对SQL型的数据库比较熟，因此对一些复杂的查询我们可以用SQL的思维来思考，再到页面[SQL to Aggregation Mapping Chart](https://docs.mongodb.org/manual/reference/sql-aggregation-comparison/)中去寻找对应的MongoDB的查询方式

###查询counter_name为hardware.memory.total时，resource_id分别为什么

* 相关操作如下：

    ```
    mongos> db.meter.aggregate([
        {
            $match: {
                counter_name: "hardware.memory.total"
            }
        },
        {
            $group: {
                _id: {
                    counter_name: "$counter_name",
                    resource_id: "$resource_id"
                }
            }
        }
    ])
    ```

###分片与复制集(Sharding与Replication)

一个完整的数据库可以备份为多份，原始的数据库和备份的数据库就组成了一个复制集，由此可以提高容错性。

一个完整的数据库的数据可以进行分片，通过分片可以把数据库中的完整数据分为多份分别存储在多台机器中，由此可以提高吞吐量。

分片和复制集是分开的两个功能，可以只做分片，也可以只做复制集。

如果既有分片又有复制集的话，那么同一个分片组成的集合就是一个复制集，如一个数据库分为两片shard1、shard2，可以再分别对shard1、shard2做两个复制shard1_1、shard1_2、shard2_1、shard2_2，那么shard1、shar1_1、shard1_2组成一个复制集，shard2、shard2_1、shard2_2组成另一个复制集。

MongoDB的每一个分片或复制集中的分片都可以不存储在同一个机器上，只要指定好IP地址和端口号即可。

本文并不讨论复制集的问题。

##MongoDB的分片，分为两片，两个分片在同一台物理机上

![MongoDB Sharding](/images/2015-11-5-Mongo-Shard/2015-11-06-MongoDB-Sharding0.png)

node-51为一台物理机，它的IP地址为172.31.2.51。

图中各服务所在IP和端口号，对应过来如下：

* shard1 --> 172.31.2.51:20000
* shard2 --> 172.31.2.51:20001
* config --> 172.31.2.51:30000
* mongos --> 172.31.2.51:27017

client通过mongos(172.31.2.51:27017)即可对数据库进行读写。

1. 新建数据目录和日志目录

    ```
    [root@node-51 ~]# mkdir -p /data/shard/s0
    [root@node-51 ~]# mkdir -p /data/shard/s1
    [root@node-51 ~]# mkdir -p /data/shard/log
    ```

2. 配置shard server

    ```
    [root@node-51 ~]# /usr/bin/mongod --shardsvr --port 20000 --dbpath /data/shard/s0 --fork --logpath /data/shard/log/s0.log --directoryperdb
    [root@node-51 ~]# /usr/bin/mongod --shardsvr --port 20001 --dbpath /data/shard/s1 --fork --logpath /data/shard/log/s1.log --directoryperdb
    ```

3. 配置config server和route server

    ```
    [root@node-51 ~]# mkdir -p /data/shard/config
    [root@node-51 ~]# /usr/bin/mongod --configsvr --port 30000 --dbpath /data/shard/config --fork --logpath /data/shard/log/config.log --directoryperdb
    [root@node-51 ~]# /usr/bin/mongos --port 27017 --configdb 172.31.2.51:30000 --fork --logpath /data/shard/log/route.log --chunkSize 1
    ```

4. admin数据库和ceilometer数据库配置

    ```
    [root@node-51 ~]# mongo admin --host 172.31.2.51 --port 27017
    mongos> use admin
    mongos> db.runCommand({addshard:'172.31.2.51:20000'})
    mongos> db.runCommand({addshard:'172.31.2.51:20001'})
    mongos> db.runCommand({enablesharding:'ceilometer'})
    mongos> db.runCommand({shardcollecton:'ceilometer.meter',key:{counter_name:1}})
    mongos> use ceilometer
    mongos> db.addUser("ceilometer","ceilometer")
    mongos> db.meter.stats()
    ```

    在这里ceilometer是一个新建的数据库，OpenStack模块的openstack-ceilometer需要连接MongoDB中的ceilometer数据库，而openstack-ceilomter在连接MongoDB中的ceilometer数据库时，使用的是用户名：ceilometer，密码：ceilometer来连接的（再安装openstack-ceilometer时设置的），所以有了db.addUser("ceilometer","ceilomter")。

5. 修改ceilometer.conf，并重启ceilometer服务

    将ceilometer.conf中的connection改为如下：

    ```
    connection=mongodb://ceilometer:ceilometer@172.31.2.51:27017/ceilometer
    ```

    重启ceilometer服务：
    
    ```
    [root@node-51 ~]# service openstack-ceilometer-alarm-evalutor restart
    [root@node-51 ~]# service openstack-ceilometer-alarm-notifier restart
    [root@node-51 ~]# service openstack-ceilometer-api restart
    [root@node-51 ~]# service openstack-ceilometer-central restart
    [root@node-51 ~]# service openstack-ceilometer-collector restart
    ```


###Mongodb分片后的开机启动设置
现在有一个问题是，设置好分片重启机器后，又得重新执行分片的命令。目前解决的办法是在/etc/rc.d/rc.local/中新增命令。

* 关闭mongod开机启动：

    ```
    [root@node-51 ~]# chkconfig --list | grep mongod --> 可以查出mongod在哪几个运行级别上运行了
    [root@node-51 ~]# chkconfig --levels 2345 mongod off
    ```

* 在文件/etc/rc.d/rc.local中，增加下述内容:

    ```
    /usr/bin/mongod --shardsvr --port 20000 --dbpath /data/shard/s0 --fork --logpath /data/shard/log/s0.log --directoryperdb
    /usr/bin/mongod --shardsvr --port 20001 --dbpath /data/shard/s1 --fork --logpath /data/shard/log/s1.log --directoryperdb
    /usr/bin/mongod --configsvr --port 30000 --dbpath /data/shard/config --fork --logpath /data/shard/log/config.log --directoryperdb
    /usr/bin/mongos --port 27017 --configdb 172.31.2.51:30000 --fork --logpath /data/shard/log/route.log --chunkSize 1

    service openstack-ceilometer-alarm-evalutor restart
    service openstack-ceilometer-alarm-notifier restart
    service openstack-ceilometer-api restart
    service openstack-ceilometer-central restart
    service openstack-ceilometer-collector restart
    ```

这个问题没算完全解决，有空再看看《鸟哥的linux私房菜》第18章 认识系统服务（daemons）和第20章 启动流程、模块管理与Loader。

##MongoDB的分片，分为三片，三个分片在不同的物理机上
这小节我会介绍一下把MongoDB中的数据库分为三片，并且把三个分片存储在不同物理机上的方法。

![Mongo-Shard](/images/2015-11-5-Mongo-Shard/2015-11-06-MongoDB-Sharding.png)

mongos1,mongos2,mongos3代表三台物理机，它们的IP地址为：

* mongos1 --> 172.31.2.135
* mongos2 --> 172.31.2.136
* mongos3 --> 172.31.2.137

图中各服务所在IP和端口号，对应过来如下：

* shard1 --> 172.31.2.135:27018
* shard2 --> 172.31.2.136:27018
* shard3 --> 172.31.2.137:27018
* config1 --> 172.31.2.135:27019
* mongos1 --> 172.31.2.135:27017

client通过连接mongos1(172.31.2.135:27017)即可对数据库进行读写。

下面详细介绍一下整个过程：

1. 安装好操作系统，安装好MongoDB，重要提醒：关闭iptables,seLinux(因为这个我中午都没睡成午觉...)

    ```
    service iptables stop
    setenforce 0
    ```

2. 在mongos1, mongos2, mongos3中新建目录
    
    ```
    [root@mongos1 ~]# mkdir -p /data/shard/s1
    [root@mongos1 ~]# mkdir -p /data/shard/log
    [root@mongos1 ~]# mkdir -p /data/shard/config

    [root@mongos2 ~]# mkdir -p /data/shard/s2
    [root@mongos2 ~]# mkdir -p /data/shard/log

    [root@mongos3 ~]# mkdir -p /data/shard/s3
    [root@mongos3 ~]# mkdir -p /data/shard/log
    ```

3. 在mongos1, mongos2, mongos3中配置shard server

    ```
    [root@mongos1 ~]# mongod --shardsvr --port 27018 --dbpath /data/shard/s1 --fork --logpath /data/shard/log/s1.log --directoryperdb

    [root@mongos2 ~]# mongod --shardsvr --port 27018 --dbpath /data/shard/s2 --fork --logpath /data/shard/log/s2.log --directoryperdb

    [root@mongos3 ~]# mongod --shardsvr --port 27018 --dbpath /data/shard/s3 --fork --logpath /data/shard/log/s3.log --directoryperdb
    ```

4. 在mongos1中配置config server

    ```
    [root@mongos1 ~]# mongod --configsvr --port 27019 --dbpath /data/shard/config --fork --logpath /data/shard/log/config.log --directoryperdb
    ```

5. 在mongos1中配置route server

    ```
    [root@mongos1 ~]# mongos --port 27017 --configdb 172.31.2.135:27019 --fork --logpath /data/shard/log/route.log --chunkSize 1
    ```

6. 在mongos1中配置admin数据库和ceilometer数据库

    ```
    [root@mongos1 ~]# mongo admin --host 172.31.2.135 --port 27017
    mongos> db.runCommand({addshard:'172.31.2.135:27018'})
    mongos> db.runCommand({addshard:'172.31.2.136:27018'})
    mongos> db.runCommand({addshard:'172.31.2.137:27018'})
    mongos> db.runCommand({enablesharding:'ceilometer'})
    mongos> db.runCommand({shardCollection:'ceilometer.meter',key:{counter_name:1}})
    mongos> use ceilometer
    mongos> db.addUser("ceilometer", "ceilometer")
    mongos> db.meter.stats()
    mongos> sh.status()
    ```

7. 修改ceilometer.conf，并重启ceilometer服务

    将ceilometer.conf中的connection改为如下：

    ```
    connection=mongodb://ceilometer:ceilometer@172.31.2.135:27017/ceilometer
    ```

    重启ceilometer服务：
    
    ```
    [root@node-51 ~]# service openstack-ceilometer-alarm-evalutor restart
    [root@node-51 ~]# service openstack-ceilometer-alarm-notifier restart
    [root@node-51 ~]# service openstack-ceilometer-api restart
    [root@node-51 ~]# service openstack-ceilometer-central restart
    [root@node-51 ~]# service openstack-ceilometer-collector restart
    ```

    可以再到mongos1中去查看数据量db.meter.find().count()，每隔一段时间执行一次，数字是不是越来越大。

##MongoDB:Expire Data from Collections by Setting TTL
当MongoDB数据库中的数据量变得很大时，查询的速度也会随之下降，定期的删除或转存数据库中的数据就成为了一个很重要的需求了。

在MongoDB 2.2中就引进了一个功能，即Expire Data from Collections by Setting TTL，有了这个功能我们只要做一个简单的设置就可以定期的删除历史数据了。

在Ceilometer的配置文件中，设置了ttl的相关参数后，Ceiloemter的后台数据库就会去自动清理数据库中的历史数据，而后台数据库不论是MongoDB还是关系型数据库都可以，当后台是MongoDB时就正是利用了MongoDB 2.2中引入的Expire Data from Collections by Setting TTL这项功能。

Ceilometer中新增自动清理数据库中的历史数据的blueprint页面为:[Database data TTL](https://blueprints.launchpad.net/ceilometer/+spec/db-ttl)，review页面为：[Database data TTL Review](https://review.openstack.org/#/c/30635/)
