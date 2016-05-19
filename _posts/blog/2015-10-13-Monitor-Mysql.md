---
layout: post
title: 一体化监控系统 - 监控云操作系统虚拟机中的MySQL数据库服务
description: OpenStack, Monitor, TopCloud, Virtual Machine, CentOS, MySQL
category: blog
---

##文档
* 监控云平台虚拟机中的数据库服务--by 房俊恒
* centos mysql安装及配置(网易云笔记)
* "一体化监控系统"安装部署指南
* [CentOS JDK Installation](http://docs.oracle.com/javase/8/docs/technotes/guides/install/linux_jdk.html#BJFJHFDD)

##云操作系统中创建租户和用户
1. 通过系统管理员(admin/admin)登录到安全云操作系统，点击系统-->租户管理-->创建租户，填写租户名称: t_dh。

2. 切换到系统安全员(safer/safer)账户，点击权限管理-->租户管理，启用新创建的租户: t_dh。

3. 切换到系统管理员(admin/admin)，点击系统-->用户管理-->创建用户:u_dh/Dh123456789。

4. 切换到系统安全员(safer/safer)账户，点击权限管理-->用户管理，启用新创建的用户:u_dh; 同时点击右侧下拉按钮，选择绑定用户，为该用户绑定一个租户(t_dh)并分配一个角色(租户管理员)。

5. 删除新创建的用户或者租户时，首先要把它们禁用。

##云操作系统中创建虚拟机
1. 创建外部网络，以便创建的虚拟机能够连接到Internet。以系统管理员(admin/admin)登录到云操作系统，点击网络-->外部网络-->创建外部网络。

2. 创建内部网络和路由器。

    以新创建的用户u_dh/Dh123456789登录到云操作系统系统，点击网络-->私有网络-->创建网络，参数如下:

    > 网络名称: private-net

    > 子网名称: dh-private-net, 网络地址: 192.168.0.0/24, IP版本: IPv4, 网关IP:空白(即缺省值)

    > 激活DHCP保持勾选状态, 分配地址池:空白, DNS域名解析服务: 61.139.2.69

    点击路由-->创建路由，把新创建的路由绑定到外部网络。

3. 创建虚拟机(云主机)，注意云主机和虚拟机表示同一个意思，后面会混用这两个词。以新创建的用户u_dh/Dh123456789登录到云操作系统，点击计算-->云主机-->创建云主机，参数如下：
    > 镜像 云主机名称: dh-host1, 镜像名称: Centos(778.3MB)

    > 规格数量 CPU: 2vCPU, 内存: 2G, 磁盘: 40G, 云主机数量: 1

    > 网络 已选择网络: private-net: dh-private-net

    > 安全 已选择的安全组: default

    > 密码/钥注入 不填

    > 确定 点击创建

    由于在创建云主机时，已经为云主机选择了前面创建的私有网络，故新创建的云主机是与私有网络直接相连的。

4. 为新创建的虚拟机分配外网IP。以新创建的用户u_dh/Dh123456789登录到云操作系统; 首先，点击网络-->外网IP-->申请外网IP，会出现"分配外网IP"的对话框，填写参数为资源池: Ext-fddf(外网名称)、数量: 1, 点击确定；然后，点击绑定到云主机。

5. 为default安全组添加规则。以新创建的用户u_dh/Dh123456789登录到云操作系统，点击网络-->安全组-->default-->添加规则，添加以下五项规则:
    > 所有TCP协议、入口

    > 所有TCP协议、出口

    > 所有ICMP协议、入口

    > 所有ICMP协议、出口

    > SSH

##虚拟机中安装和配置MySQL
1. 安装MySQL
    
    ```
    yum list mysql-server
    yum install mysql-server
    ```

2. 设置MySQL的服务

    启动MySQL服务: 
    
    ```
    service mysqld start
    ```

    连接MySQL并退出: 
    
    ```
    mysql --> \q
    ```

    设置MySQL开机启动:
    
    ```
    chkconfig mysqld on
    chkconfig list
    chkconfig --list
    ```
    
    开启3306端口并保存:
    
    ```
    /sbin/iptables -I INPUT -p tcp --dport 3306 -j ACCEPT
    /etc/rc.d/init.d/iptables save
    ```

3. 修改密码并设置远程访问

    连接MySQL数据库，并设置密码：

    ```
    mysql
    use mysql;
    update user set password=password('123456') where user='root';
    flush privileges;
    ```

    设置MySQL远程访问:

    ```
    grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;
    ```

4. 解决MySQL乱码问题

    在没有my.cnf时需要找一个配置文件，复制到/etc/目录，命名为my.cnf

    ```
    find / -iname "*.cnf" -print
    ```

    修改/etc/my.cnf, 在[client]和[mysqld]下面添加: default-character-set=utf8

5. 重启MySQL服务

    ```
    service mysqld restart
    ```

##虚拟机中安装iRadar Agent

1. 在Windows下通过浏览器进入一体化监控系统的WEB Portal，在监控客户端下载界面根据操作系统平台下载对应的iRadar Agent安装包文件和授权文件。

    ```
    iRadar Agent安装包文件: iradar-agent-1u02-rh3l.6.x.bin
    授权文件: iradar_agentd.license
    ```

2. 使用SSH Secure Shell将安装文件和授权文件放入虚拟机的/home目录下

3. 安装iRadar Agent

    ```
    cd /home
    chmod 777 iradar-agent-1u02-rh31.6.x.bin
    ./iradar-agent-1u02-rh31.6.x.bin install
    ```

4. 覆盖/etc/iradar/目录下的授权文件

    ```
    cd /home
    cp iradar_agentd.license /etc/iradar/iradar_agentd.license
    ```

5. 启动iradar agent服务

    ```
    service iradar-agent start
    ```

6. 在虚拟机中安装jdk

    从内网中下载获得JDK的RPM安装包: jdk-8u11-linux-x64.rpm，并通过SSH Secure Shell拷贝到虚拟机的/home目录下

    安装JDK，可参考官方文档[CentOS JDK Installation](docs.oracle.com/javase/8/docs/technotes/guides/install/linux_jdk.html#BJFJHFDD)

    ```
    rpm -ivh jdk-8ull-linux-x64.rpm
    ```

##一体化监控系统中添加数据库监控对象

1. 一体化监控系统支持对数据库的监控，监控数据库的前提操作配置：

    > 在数据库所在主机正确安装一体化监控系统iRadar Agent软件，并启动服务

    > 在数据库所在主机安装Java环境(JDK或JRE运行环境)

    > 数据库要授予相关用户的远程、本地连接权限

2. 通过浏览器进入一体化监控系统WEB Portal

3. 依次点击设备中心-->设备监控配置-->添加设备进入设备添加界面，完成设备各属性填写，在预定义宏选项卡中，添加五个宏变量：

    > {$USER}: 数据库用户名(root)

    > {$PSWD}: 数据库密码(123456)

    > {$DBIP}: 数据库所在主机IP地址(172.31.2.105，这是虚拟机的外网IP地址)

    > {$PORT}: 数据库端口号(3306)

    > {$JAVAHOME}: 数据库所在主机的JDK或JRE安装目录(/usr/java/jdk1.8.0_11)
