---
layout: post
title: 一体化监控系统 -- 云操作系统中创建虚拟机时分配的磁盘空间与虚拟机中看到的磁盘空间大小不一
description: OpenStack, Image, Disk Size
category: blog
---

##文档
* [OpenStack Image Guide Ch3](http://docs.openstack.org/image-guide/content/ch_openstack_images.html)
* [Extending an LVM volume](https://www.turnkeylinux.org/blog/extending-lvm)
* [LVM Administrator's Guide](https://www.centos.org/docs/5/html/Cluster_Logical_Volume_Manager/)
* [archlinux LVM](https://wiki.archlinux.org/index.php/LVM)
* [CentOS 6.3下配置LVM](http://cnblogs.com/mchina/p/linux-centos-logical-volume-manager-lvm.html)
* LVM: Logical Volume Manager

##需要使用到的Linux命令
* parted
* pvcreate
* vgextend
* lvextend
* resize2fs

##环境
* 云安全操作系统1.0
* 镜像规格
    > 规格 778.3MB

    > 容器格式 BARE

    > 磁盘格式 QCOW2

    > 最小磁盘 10GB

    > 最小内存 512MB

##问题描述
1. 使用链接[云操作系统中创建虚拟机](http://xiaofandh12.github.io/Monitor-Mysql)中的方法创建虚拟机dh-test，下图是该虚拟机的详细信息，在图中可以看到磁盘的大小为40G。
![dh-test info](/images/2015-10-14-VirtualMachine-Disk-Resize/dh-test.jpg)

2. 登录虚拟机dh-test，使用命令: df -hl，结果截图如下:
![dh-test df -hl](/images/2015-10-14-VirtualMachine-Disk-Resize/dh-test-df.jpg)

    可以看到能使用的空间完全不到40G，差得很远呀。

3. 登录虚拟机dh-test，使用命令: fdisk -l，结果截图如下，/dev/vda中的很大一部分空间没有利用:
![dh-test fdisk -l](/images/2015-10-14-VirtualMachine-Disk-Resize/dh-test-fdisk.jpg)

    > Disk /dev/vda: 42.9GB

    > Disk /dev/mapper/vg_centos6-lv_root: 2302MB

    > Disk /dev/mapper/vg_centos6-lv_swap: 314MB

##问题解决
1. 安装工具parted

   ```
   yum -y install parted
   ```

2. 查看设备/dev/vda已有的分区信息

    ```
    parted /dev/vda print 或
    fdisk -l /dev/vda
    ```

    可以看到有了/dev/vda1, /dev/vda2两个分区

3. 查看设备/dev/vda的信息

    ```
    parted /dev/vda --script "print free"
    ```

    可以看到有从3146MB到42.9GB共39.8GB的Free Space

4. 利用设备/dev/vda中的空闲空间创建一个主分区

    ```
    parted /dev/vda mkpart primary ext4 3146MB 42.9GB
    shutdown -r now (重启以使创建主分区生效)
    ```

    新创建的分区名为/dev/vda3(因为前面有了2个分区)

5. 创建一个物理分区(Physical Volume)

    ```
    pvcreate /dev/vda3
    ```

6. 将新创建的物理分区(Physical Volume)加入一个卷组(Volume Group)

    ```
    vgdisplay --> 可以查出卷组名称为: vg_centos6
    vgextend vg_centos6 /dev/vda3
    ```

7. 使用/dev/vda3扩展root的逻辑分区

    ```
    lvextend /dev/mapper/vg_centos6-lv_root /dev/vda3
    ```

    root的逻辑分区名称为：lv_root，可以使用df -hl查出; vg_centos6-lv_root中卷组名称为vg_centos6，root逻辑分区为：lv_root

8. Resize the root file system

    ```
    resize2fs /dev/mapper/vg_centos6-lv_root
    ```

完成

