---
layout: post
title: 逻辑卷管理器（Logical Volume Manager, LVM）
description: LVM
category: blog
---

## 参考文档

* [鸟哥的Linux私房菜-第3章 主机规划与磁盘分区](http://vbird.dic.ksu.edu.tw/linux_basic/0130designlinux.php)

* [鸟哥的Linux私房菜-第8章 Linux磁盘与文件系统管理](http://vbird.dic.ksu.edu.tw/linux_basic/0230filesystem.php)

* [鸟哥的Linux私房菜-第15章 15.3 逻辑卷管理器](http://vbird.dic.ksu.edu.tw/linux_basic/0420quota_3.php)

* [云操作系统中创建虚拟机时分配的磁盘空间与虚拟机中看到的磁盘空间大小不一](http://xiaofandh12.github.io/VirtualMachine-Disk-Resize)

* [A Beginner's Guide To LVM - HowtoForge](https://www.howtoforge.com/linux_lvm)

* [LVM Administor's Guide - CentOS](https://www.centos.org/docs/5/html/Cluster_Logical_Volume_Manager/)

* [Linux lvm - 	Logical Volume Manager](https://linuxconfig.org/linux-lvm-logical-volume-manager)

* [How to Manage and Use LVM (Logical Volume Management) in Ubuntu](http://www.howtogeek.com/howto/40702/how-to-manage-and-use-lvm-logical-volume-management-in-ubuntu/)

* [wiki-Logical Volume Manager(Linux)](https://en.wikipedia.org/wiki/Logical_Volume_Manager_\(Linux\))

* [LVM HOWTO - The Linux Documentation Project](http://tldp.org/HOWTO/LVM-HOWTO/)

* [LVM Resource Page](https://sourceware.org/lvm2/)

* man lvm

## PV, PE, VG, LV

![LVM各组件的实现流程图示](/images/2016-05-18-Logical-Volume-Manager-LVM/lvm.gif)

* PV,Physical Volume,物理卷

* VG,Volume Group,卷用户组

* LV,Logical Volume,逻辑卷

* PE,Physical Extend,物理扩展块

* PE与VG的相关性图示
* 
![PE and VG](/images/2016-05-18-Logical-Volume-Manager-LVM/pe_vg.gif)



## 知识点
* LVM必须要有内核支持且需要安装lvm2这个软件，一般的发现版Linux都把LVM的内核支持与软件安装好了。
* 一个磁盘要想用起来有两种方法：
	* 磁盘分区 --> 磁盘分区格式化 --> 挂载
	* 磁盘分区 --> PV阶段(创建PV) --> VG阶段(创建VG或扩展VG) --> LV阶段(创建逻辑卷或扩展逻辑卷) --> 文件系统阶段(格式化、挂载或扩展文件系统)
* LVM的常见使用：
	* 放大LV容量
	* 缩小LV容量
	* LVM的系统快照
	* 删除系统内的LVM

