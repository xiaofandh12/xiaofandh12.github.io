---
layout: post
title: Zabbix Server源码分析之概貌介绍
description: Zabbix,Zabbix Server,数据库,进程
category: blog
---

## 文档

* [zabbix2.2 doc](http://www.zabbix.com/documentation/2.2/start)
* [zabbix源码阅读－－zabbix_server](http://blog.csdn.net/liujian0616/article/details/7946492)
* [zabbix数据库表结构－持续更新](http://www.furion.info/623.html)
* [ZABBIX数据库结构详解(一):HOSTS表](http://bbs.linuxtone.org/thread-20874-1-1.html)

## Zabbix术语

### __Host相关__：

* __host group__: a logical grouping of hosts; it may contain hosts and templates. Hosts and templates within a host group are not in any way linked to each other. Host groups are used when assigning access rights to hosts for different user groups.

* __host__: a networked device that you want to monitor, with IP/DNS.

* __template__: a set of entities (items, triggers, graphs, screens, applications, low-level discovery rules, web scenarios) ready to be applied to one or several hosts. The job of templates is to speed up the deployment of monitoring tasks on a host; also to make it easier to apply mass changes to monitoring tasks. Templates are linked directly to individual hosts.

* __application__: a grouping of items in a logical group.

* __item__: a particular piece of data that you want to receive off a host, a metric of data.

* __trigger__: a logical expression that defines a problem threshold and is used to "evaluate" data received in items. When received data are above the threshold, triggers go from 'OK' into 'Problem' state. When received data are blow the threshold, triggers stay in/return to an 'OK' state.

* __graph__: There are two kinds of graph: Simple graphs, Custom graphs. Simple graphs are provided for the visualization of data gathered by items. No configuration effort is required on the user part to view simple graphs. They are freely made available by Zabbix. Custom graphs, as the name suggests, offer customisation capabilities. While simple graphs are good for viewing data of a single item, they do not offer configuration capabilities. Thus, if you want to change graph
  style or the way lines are displayed or compare several items, for example incoming and outgoing traffic in a single graph, you need a custom graph. Custom graphs are configured manually. They can be created for a host or several hosts or for a single template. 

* __discovery__: Zabbix offers automatic network discovery functionality that is effective and very flexible; It is possible to allow active Zabbix agent auto-registration, after which the server can start monitoring them; Low-level discovery provides a way to automatically create items, triggers, and graphs for different entities on a computer.

* __web scenario__: one or several HTTP requests to check the availability of a web site.

* __interface__: 

### __告警相关__：

* __event__: a single occurrence of something that deserves attention such as a trigger changing state or a discovery/agent auto-registration taking place.

* __action__: a predefined means of reaching to an event. An action consistes of operations (e.g. sending a notification) and conditions (when the operation is carried out).

* __escalation__: a custom scenario for executing operations within an action; a sequence of sending notifications/executing remote commands.

* __media__: a means of delivering notifications; delivery channel.

* __notification__: a message about some event sent to a user via the chosen media channel.

* __remote command__: a pre-defined command that is automatically executed on a monitoted host upon some condition.

## Zabbix Server配置文件与Zabbix Agent配置文件

### zabbix_server.conf

官方参考文档:[Daemon configuration: zabbix server](https://www.zabbix.com/documentation/2.2/manual/appendix/config/zabbix_server)

### zabbix_agentd.conf

官方参考文档:[Daemon configuration: zabbix agent(UNIX)](https://www.zabbix.com/documentation/2.2/manual/appendix/config/zabbix_agentd)

官方参考文档:[Daemon configuration: zabbix agent(Windows)](https://www.zabbix.com/documentation/2.2/manual/appendix/config/zabbix_agentd_win)

### zabbix_proxy.conf

官方参考文档:[Daemon configuration: zabbix proxy](https://www.zabbix.com/documentation/2.2/manual/appendix/config/zabbix_proxy)

## Zabbix Server数据库分析

下图是zabbix前端中Configuration --> Hosts的信息，从中可以看到我们能够为一个Host配置Applications, Items, Triggers, Graphs, Discovery, Web, Interface, Templates.
![Zabbix Host Info](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Host-Info.png)

下面列出了与Host配置(Host, Applications,Items,Triggers,Graphs,Discovery,Web,Interface)相关的数据库表格、监控项目（Item）历史数据相关的数据库表格、用户信息相关的数据库表格。下面我只会介绍某一大类数据库表格下一些重要表格的结构、作用，以及重要表格之间的连接关系，而对其它的数据库表格只给出了数据库表名。

### Host相关表格:

* hosts

在zabbix中host和template有着不同的定义（见前面zabbix术语部分），但它们都包含Applications、Items、Triggers、Graphs、Screens、Discovery、Web、Linked templates（templates），可以说template就是zabbix事先定义好的host，所以在数据库的存储方面host和template的信息都存储在hosts表格中。
hosts表格及其重要，它记录了host几乎所有的信息，而其它比较重要的表格（applications、items、interface、httptest）是通过hosts表格中的hostid来关联上hosts表格的。

![Zabbix Server Database hosts](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-hosts.png)

* hosts_groups

* host_inventory

* hostmacro

* hosts_templates

hosts_templates表格给出了host与template之间的链接关系。

![Zabbix Server Database hosts_templates](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-hosts_templates.png)

* groups

### Applications相关表格:

application是一个或多个item的集合，一个item可以属于一个或多个application。
一个template(host)可以包含一个或多个application，一个application可以属于一个或多个template(host)。

* applications

applications表格通过hostid关联了hosts表格，并指明了每个applicationid的name。

![Zabbix Server Database applications](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-applications.png)

* application_template

没弄懂？

![Zabbix Server Database application_template](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-application_template.png)

* items_applications

items_applications表格指明了一个application包含了那些item。

![Zabbix Server Database items_applications](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-items_applications.png)

### Items相关表格:

* items

items表格通过hostid关联了hosts表格，并有每个item的详细信息。

![Zabbix Server Database items](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-items.png)

### Triggers相关表格:

以下的表格都和告警相关

* triggers

triggers表格包含了所有触发器的详细信息。

![Zabbix Server Database triggers](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-triggers.png)

* functions

functions表格通过itemid,triggerid将items表格和triggers表格关联了起来，并包含function的其它信息。

![Zabbix Server Database functions](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-functions.png)

* trigger_depends

* actions

* alerts

* events

* media

* media_type

### Graphs相关表格:

* graphs

graphs表格包含了zabbix中图形(graph)的详细信息。

![Zabbix Server Database graphs](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-graphs.png)

* graphs_items

graphs_items表格包含graphid、itemid两个字段，graphs和items就是通过这两个字段关联起来的。

![Zabbix Server Database graphs_items](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-graphs_items.png)

* graph_theme

### Discovery相关表格:

* graph_discovery

* group_discovery

* host_discovery

![Zabbix Server Database host_discovery](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-host_discovery.png)

* interface_discovery

* item_discovery

* trigger_discovery

### Web相关表格:

* httpstep

* httpstepitem

* httptest

httptest表通过hostid与hosts表格关联，并包含web监控的其它相关信息。

![Zabbix Server Database httptest](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-httptest.png)

* httptestitem

### Interface相关表格:

* interface

interface表通过hostid与hosts表格关联，并包含该host中代理的ip、端口号等。

![Zabbix Server Database interface](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-interface.png)

### Item history info相关表格:

* history

history表保存了zabbix server从各途径采集到的各监控项目(item)的历史数据，history_log、history_str、history_text、history_uint都是保存监控项目历史数据的，只是数据类型不同而已。

![Zabbix Server Database history](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-history.png)

* history_log

* history_str

* history_str_sync

* history_sync

* history_text

* history_uint

* history_uint_sync

### Trend相关表格:

* trends

trends表保存了各监控项目(item)的趋势数据。

![Zabbix Server Database trends](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-trends.png)

* trends_uint

### 用户信息相关表格:

* users

users表保存了zabbix中的用户信息。

![Zabbix Server Database users](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Database-users.png)

* user_history

* users_groups

* usrgrp

* config

* profiles

* rights

* screens

* screens_items

* sessions

## Zabbix Server各进程及其功能分析

Zabbix2.2.11 共有17个进程，它们大致的作用和关系如下图所示：

![Zabbix Server Process Architecture](/images/2015-12-18-Zabbix-Server/2015-12-18-Zabbix-Server-Process-Architecture.png)

下面介绍各进程的功能，我会以如下的格式来说明各进程的功能：
    
### 进程名：
        
> 源码中的注释；
        
> 我自己的理解。

### main_dbconfig_loop():
    
> periodically synchronises database data with memory cache;

### main_discoverer_loop():
        
> periodically try to find new hosts and services;

### main_trapper_loop():
        
> periodically try to find new hosts and services;

### main_snmptrapper_loop():

> SNMP trap reader's entry point;

### main_poller_loop():
        
> None;

### main_proxypoller_loop():

> None;

### main_httppoller_loop():
        
> main loop of processing of httptest;

### main_vmware_loop():

> the vmware collector main loop;

### main_dbsyncer_loop():
        
> periodically synchronises data in memory cache with database;

### main_housekeeper_loop():

> None;

### main_timer_loop():
        
> periodically updates time-related triggers;

### main_alerter_loop():
        
> periodically check table alerts and send notifications if needed;

### main_escalator_loop():
        
> periodically check table escalations and generate alerts;

### main_watchdog_loop():
        
> check database availability every DB_PING_FREQUENCY seconds and alert admins if it is down;

### main_pinger_loop():
        
> periodically perform ICMP pings;

### main_nodewatcher_loop():
        
> periodcally calculates checksum of config data;

### main_selfmon_loop():
> None;
