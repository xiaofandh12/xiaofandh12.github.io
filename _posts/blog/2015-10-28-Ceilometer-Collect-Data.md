---
layout: post
title: OpenStack Ceilometer - 数据采集原理
description: OpenStack,Ceilometer
category: blog
---
##文档
* [openstack telemetry admin guide](http://docs.openstack.org/admin-guide-cloud/telemetry.html)
* [openstack ceilometer developer guide](http://docs.openstack.org/developer/ceilometer)
* [openstack ceilometer config](http://docs.openstack.org/kilo/config-reference/content/ch_configuring-openstack-telemetry.html)
* [openstack telemetry api v2](http://developer.openstack.org/api-ref-telemetry-v2.html#telemetry-v2)
* 《OpenStack设计与实现》- 第9章 计量与监控
* 《OpenStack设计与实现》- 第4章 OpenStack通用技术
* 《OpenStack开源云》- 第11章 OpenStack服务分析

##Ceilometer的定位
###定位
![Ceilometer Position](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Position.png)

图中iradar_agent和监控系统目前属于一体化监控系统，iradar_agent是收集数据的代理，监控系统是前端展示界面。

Ceilometer和iradar_agent都会收集数据，作为Heat的决策基础，并为Horizon提供数据支持。
我们这里是将Ceilometer收集到的数据，存储在Mongodb中。
###问题
Ceilometer优化问题：

* Ceilometer和iradar_agent该如何分工
* Ceilometer该收集哪些数据以及Polling Agent该以什么频率去轮询
* 如何在Ceilometer中新增插件、告警去定期清理Mongodb中的历史数据
* 采用Mongodb提供的分布式存储能否提高效率，[Sileht](https://github.com/sileht)的一篇博客中介绍了在[Ceilometer中使用sharding/replicaSet mongodb的方法](https://blog.sileht.net/using-a-shardingreplicaset-mongodb-with-ceilometer.html)，Sileht是ceilometer的一个[Contributor](https://github.com/openstack/ceilometer/graphs/contributors/)

Horizon集成问题：

* Horizon与计费系统的集成
* Horizon与监控系统的集成
* Horizon与数据分析的集成

##Ceilometer大体概述
Ceilometer模块的目的：

* 为了获取并保存计量信息来支持对用户的收费
* 成为OpenStack系统里一个标准的获取并保存测量值的框架
* 利用已保存的测量值进行报警

Ceilometer通过如下三种方式获取测量值数据：

* 第三方的数据发送者把数据以通知信息(Notification Message)的形式发送到消息总线(Notification Bus)上，Ceilometer中的Notification Agent会获取这些通知事件，从中提取测量数据
* Ceilometer中的Polling Agent会根据配置，定期主动通过各种API或其它协议去远端或者本地的不同服务实体中获取所需的测量数据
* 用户可以通过调用Ceilometer RESTful API直接把利用其它方式获取的任意测量数据送达给Ceilometer（这种方式没在下面的Ceilometer数据收集原理中说明）

三种从Ceilometer获取数据的方式：

* with the RESTful API
* with the command line interface
* with the Metering tab on an openstack dashboard

Ceilometer的服务：

* ceilometer-api
* ceilometer-polling
* ceilometer-agent-central
* ceilometer-agent-compute
* ceilometer-agent-ipmi
* ceilometer-agent-notification
* ceilometer-collector
* ceilometer-alarm-evaluator
* ceilometer-alarm-notifier

Ceilometer服务也时常在变，下图是Juno版本setup.cfg中console scripts的定义:
![Ceilometer Console Scripts Juno](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Console-Scripts-Juno.jpg)

下图是Liberty版本setup.cfg中console scripts的定义：
![Ceilometer Console Scripts Liberty](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Console-Scripts-Liberty.jpg)

##Ceilometer数据采集原理
下图是我看了资料后画出来的，图是用Inkscape画的，要改箭头的颜色需要下一个Inkscape的插件，太麻烦，我忍了。
注意，这部分内容没有涉及Event和Alarm，不是它们不重要，而是图上面画不下。
![Ceilometer Collect Data](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Collect-Data.png)

下面我会对图中的一些关键点做一些解释。

###采集数据的第一种方式
图中粉红色的线条表示第一种数据收集方式，Compute(nova compute)、Volume(swift)、Network(nutron)、Image(glance)、Object storage(cinder)等模块的服务将会把一些数据主动发送到消息总线(Notification Bus)上，当消息总线(Notification Bus)上出现了Notification Agent感兴趣的数据时，Notification Agent就会通过Notification Listener获取该数据，并转换为Sample发给Pipeline做下一步处理，在这个数据传输的过程中用到了OpenStack的通用库oslo.messagine。

Notificaiton Listener插件的entroy points定义在setup.cfg文件中的ceilometer.notification，截图如下：
![Ceilometer Notification](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Notification.jpg)

Notification Agent的运行流程如下：

* 解析Pipeline配置文件得到的Notification Listener的定义
* 调用stevedore库，载入所有的Notification Listener插件
* 对每一个Notification Listener插件，通过oslo.messaging库构造其对应的oslo.messaging的Notification Listener对象，并且启动此对象监听通知消息(Notification Message)

###采集数据的第二种方式
图中蓝色的线条表示另一种数据收集方式，Polling Agent获取数据的方式是定期调用不同的Pollster插件去轮询Compute(nova compute)、Volume(swift)、Network(nutron)、Image(glance)、Object stroage(cinder)、其它设备，在这里数据传输方式可以为SNMP或RESTful API，Polling Agent会把获取到的数据转换为Sample发给Pipeline做下一步处理。

Polling Agent包括Compute Agent、Central Agent、IPMI Agent，Compute Agent需要部署在运行Nova Compute服务的计算节点上，它主要是用来和Hypervisor进行通信的，轮询获取Hypervisor相关的测量值；Central Agent可以被部署在任何节点上，它用来和远程的各种不同的实体和服务进行通信，获取不同的测量值；IPMI Agent需要部署在IPMI的节点上，用来获取本机IPMI的相关测量值。

Compute Agent、Central Agent、IPMI Agent的Pollster插件定义在setup.cfg文件中的ceilometer.poll.compute、ceilometer.poll.central、ceilometer.poll.ipmi,截图如下：
![Ceilometer Poll Compute](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Poll-Compute.jpg)
![Ceilometer Poll Central](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Poll-Central.jpg)
![Ceilometer Poll ipmi](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Poll-ipmi.jpg)

各种Polling Agent的运行流程基本比较类似：

* 调用stevedore库，获取属于本Agent的所有Pollster插件
* 创建PartitionCoordinator类实例对象，加入一个partition group，并创建一个定时器用以周期性地发送心跳信息
* 解析Pipeline配置文件得到Pipeline的定义，并根据解析结果创建一个或者多个不同的PollingTask和所对应的定时器。由于所有采样频率相同的Pipeline都会在同一个PollingTask里处理，所以每个PollingTask都由某个特定频率的定时器驱动，在某一个协程中被执行。
* 由定时器驱动的PollingTask会周期性地调用其所包括的各种Pollster,由这些Pollster获取测量数据，然后根据Pipeline的定义，把获取的测量取样值交给Transformer转换再由Publisher发布。

###获取到Sample后的处理
获取到Sample对象后，Sample会交给Pipeline，由Transformer转换再由Publisher发布，关于这一步是在pipeline.yaml文件中配置。Transformer和Publisher都有多种，定义在setup.cfg中，截图如下：
![Ceilometer Transformer](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Transformer.jpg)
![Ceilometer Publisher](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Publisher.jpg)

Collector会通过AMQP(RabbitMQ)来获取Publisher发布的数据，然后交给不同的Dispathcher，根据Dispatcher的不同把数据保存在不同的地方(Database,File,http,Gnocchi)。Dispathcher也定义在setup.cfg中，截图如下：
![Ceilometer Dispatcher](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-Dispatcher.jpg)

###Inspector

##openstack-ceilometerclient
在安装了openstack-ceilometerclient服务的机器上，我们可以运行ceilometer命令。通过ceilometer命令我们可以学着调用Telemetry API v2。

1. 运行ceilometer --debug meter-list
![Ceilometer Debug](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-debug.jpg)

2. 把上图中curl命令那一行复制下来，照着[openstack telemetry api v2](http://developer.openstack.org/api-ref-telemetry-v2.html#telemetry-v2)修
改下就可以查询或修改其它的值，如查询alarms：
![Ceilometer curl](/images/2015-10-28-Ceilometer-Collect-Data/2015-10-28-Ceilometer-curl.jpg)
因为没有alarm所以返回的数据体为空。
