---
layout: post
title: OpenStack Ceilometer -- MongoDB的监控工具mongostat与MongoDB Cloud Manager
description: OpenStack,Ceilometer,MongoDB,mongostat,MongoDB Cloud Manager
category: blog
---

## 文档

* [mongostat doc](https://docs.mongodb.org/manual/reference/program/mongostat/)
* [MongoDB Cloud Manager](https://cloud.mongodb.com/)
* [MongoDB Cloud Manager Doc](https://docs.cloud.mongodb.com/?_ga=1.54557739.1477311058.1443419873)
* [mtools](https://github.com/rueckstiess/mtools)

## mongostat

在安装好MongoDB的机器上，我们可以使用mongostat命令来监控和查看MongoDB的一些实时数据，下图是我机器运行的一个例子：

![mongostat](/images/2015-11-13-MongoDB-Monitoring/2015-11-13-mongostat.jpg)

图中参数的意义以及mongostat命令的选项，可以通过mongostati --help来查看。

## MongoDB Cloud Manager

MongoDB Cloud Manager是一个在线的监控MongoDB的工具，它的服务是部署在AWS上的。
MongoDB Cloud Manager非常容易使用，主要是网站的引导做得相当好，基本不用看什么文档，跟着它步骤来就可以了。
大概步骤如下：

    > 注册MongoDB

    > Delopyment --> Deployment --> +ADD --> Existing MongoDB Deployment

    > 下载agent，安装agent

    > 验证agent可用

    > 填写host信息，我用了分片，但也只用输入mongos在的host信息

下图是我配置好后在网站上看到的信息

![MongoDB Cloud Manager 1](/images/2015-11-13-MongoDB-Monitoring/2015-11-13-MongoDB-Cloud-Manager1.jpg)

因为我截图的时候，ceilometer已经没有向mongos1:27017里写入数据了，故这里的线都比较直。
![MongoDB Cloud Manager 2](/images/2015-11-13-MongoDB-Monitoring/2015-11-13-MongoDB-Cloud-Manager2.jpg)

## mtools

mtools is a collection of helper scripts to parse and filter MongoDB log files(mongod, mongos), visualize log files and quickly set up complex MongoDB test environments on a local machine.

它包含的工具包括：mlogfilter, mloginfo, mplotqueries, mlogvis, mlaunch, mgenerate。

mlaunch is a script to quickly spin up local test environments, including replica sets and sharded systems(requires pymongo).这个工具比较好，不用我再手动去分片了，不过只能在单机上用，实际意义就不大了。

