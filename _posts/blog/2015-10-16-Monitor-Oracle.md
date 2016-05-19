---
layout: post
title: 一体化监控系统: 监控云操作系统虚拟机中的Oracle数据库服务
description: OpenStack, Monitor, TopCloud, Virtual Machine, CentOS 6.5, Oracle
category: blog
---

##云操作系统中创建虚拟机
参看[使用一体化监控系统监控云操作系统虚拟机中的MySQL数据库服务](https://xiaofandh12.github.io/Monitor-Mysql/)的云操作系统中创建虚拟机部分

##安装常用工具
* vim: yum -y install vim
* unzip: yum -y install unzip
* 配置vim: https://github.com/amix/vimrc
* lsb_release命令：yum -y install redhat-lsb

##调整虚拟机中磁盘大小
参看[云操作系统中创建虚拟机时分配的磁盘空间与虚拟机中看到的磁盘空间大小不一](https://xiaofandh12.github.io/VirtualMachine-Disk-Resize/)

##安装JDK
在官方文档中，有说明安装Oracle前安装JDK是一个可选项，但是不安装JDK总出错，反正最后一体化监控系统也要求在虚拟机中安装JDK，所以索性先装上，安装JDK可参看[CentOS JDK Installation](http://docs.oracle.com/javase/8/docs/technotes/guides/install/linux_jdk.html#BJFJHFDD)

##在虚拟机中安装Oracle数据库

###注意事项
1. 下面的文档都应该看看，尤其是官方文档，文档之间可以结合起来看，每个文档或多或少都有可能出错，包括我写的这篇文档
2. 在前期操作系统环境配置的过程中，[官方文档 Oracle Database Preinstallation Tasks](http://docs.oracle.com/cd/E11882_01/install.112/e47689/pre_install.htm#LADI1085)这里的说法最权威。
3. 前期操作系统环境配置的项目比较多，慢点儿来，仔细看
4. 安装过程中的日志文件对我们定位问题有很大帮助
5. 在Oracle数据库软件安装好后，还要创建数据库和配置监听

###文档
* [官方文档 Oracle Database Installation Guide for Linux](http://docs.oracle.com/cd/E11882_01/install.112/e47689/toc.htm)
* [官方文档 Oracle Database Preinstallation Tasks](http://docs.oracle.com/cd/E11882_01/install.112/e47689/pre_install.htm#LADI1085)
* [官方文档 Installing Oracle Database](http://docs.oracle.com/cd/E11882_01/install.112/e47689/inst_task.htm#LADI1242)
* [官方文档 Installing and COnfiguring Oracle Database Using Response Files](http://docs.oracle.com/cd/E11882_01/install.112/e47689/app_nonint.htm#LADBI1341)
* [Oracle 11g R1/R2官方下载列表](http://www.linuxidc.com/Linux/2012-04/59064.htm)
* [自己动手CentOS-6.5安装Oracle11g R2,我最主要参考的文档](http://blog.itpub.net/29742691/viewspace-1214803/)
* [CentOS下无界面静默安装oracle 11g](http://wiselyman.iteye.com/blog/2115934)
* [Oracle 11g静默安装step by step](http://blog.sina.com.cn/s/blog_3eb222740100ij71.html)
* [RHEL6(CentOS6)安装Oracle 11g R2手记](http://blog.csdn.net/kimsoft/article/details/8117575/)
* [Oracle11G基于CentOS6.4静默安装](http://blog.itpub.net/29025273/viewspace-1258114)
* [Oracle静默安装文件db_install.rsp详解](http://blog.csdn.net/alexdream/article/details/16862339/)

###硬件环境
* 操作系统：CentOS 6.6 x86 x64 minimal.iso
* 主机名：oracledb
* 内存：2G
* 硬盘：40G
* Oracle数据文件：linux.x64_11gR2_database_1of2.zip linux.x64_11gR2_database_2of2.zip

云操作系统中虚拟机的基本信息：
![oracledb.jpg](/images/2015-10-16-Monitor-Oracle/oracledb.jpg)

虚拟机中操作系统详情：
![lsb_release.jpg](/images/2015-10-16-Monitor-Oracle/lsb_release.jpg)

###操作系统环境设置

1. 修改操作系统
    
    ```
    [root@oracledb ~]# vim /etc/redhat-release 修改文件/etc/redhat-release的内容
    #CentOS release 6.5 (Final) 将这行注释掉
    Red Hat Enterprise Linux 6
    ```

2. 修改主机名

    ```
    [root@oracledb ~]# sed -i "s/HOSTNAME=localhost.localdomain/HOSTNAME=oracledb/" /etc/sysconfig/network
    [root@oracledb ~]# hostname oracledb
    ```

3. 添加主机名与IP对应记录

    ```
    [root@oracledb ~]# vi /etc/hosts 在文件/etc/hosts中添加下面这一行内容
    172.31.2.93 oracledb
    [root@oracledb ~]# ping oracledb 能ping通即代表成功
    ```

4. 安装依赖包

    官方文档中要求安装的依赖包如下图，下图列出的是Oracle Database Package Requirements for Linux x86-64,要把能安装的i686相关的包都安装上:
    ![PackageRequirements](/images/2015-10-16-Monitor-Oracle/PackageRequirements.jpg)

    安装了这些包后，在后面安装过程的日志中可以看到如下图的错误，下图是通过命令```grep --color "Error" installActions2015-10-19_10-53-18AM.log```，其中installActions2015-10-19_10-53-18AM.log为安装过程中的日志文件：
    ![LogError](/images/2015-10-16-Monitor-Oracle/LogError.jpg)

    在Red Hat Enterprise Linux 4和Red Hat Enterprise Linux 5中要求安装elfutils-libelf-devel-0.97，但是在Red Hat Enterprise Linux 6中我们可以看到并没有要求安装elfutils-libelf-devel-0.97。

    那几个由于没有装i386需要的包，而产生的错误，完全可以忽略。

    unixODBC-2.2.11 (x86_64)和unixODBC-devel-2.2.11 (x86_64)是需要用到ODBC时才需要安装的包，不装也没有问题。

    事实上，我们查询安装过程中的日志文件installActions2015-10-19_10-53-18AM.log，我们可以看到上图产生的错误都是可以IGNORABLE的，如我们可以在日志文件中查询某一个错误，截图如下：
    ![ErrorIgnorable](/images/2015-10-16-Monitor-Oracle/ErrorIgnorable.jpg)

    从中可以看到Severity:IGNORABLE字样，说明该错误是可以忽略的。不过为了少看到点儿错误我们可以把elfutils-libelf-devel-0.97,unixODBC-2.2.11和unixODBC-devel这几个包安装上，反正安装过程相当简单，一行命令的事儿。

4. 创建用户oracle和组oinstall,dba
    
    ```
    [root@oracledb ~]# groupadd -g 251 oinstall
    [root@oracledb ~]# groupadd -g 252 dba
    [root@oracledb ~]# useradd -u 256 -g oinstall -G dba -d /opt/oracle -s /bin/bash -m oracle
    [root@oracledb ~]# passwd oracle
    ```

    > -g: 指定用户所属的群组
    
    > -G: 指定用户所属的附加群组

    > -u: 指定用户ID

    > -d: 指定用户主目录

    > -s: 指定用户登录shell

    > -m: 若用户主目录不存在，则自动创建

    > 如果最后修改设定密码为oracle，系统会提示密码过于简单，此时无需理会，再次输入回车即可

5. 修改内核参数

    ```
    [root@oracledb ~]# vim /etc/sysctl.conf 在文件末尾添加如下几行
    fs.aio-max-nr = 1048576
    fs.file-max = 6815744
    kernel.shmall = 2097152
    kernel.shmmax = 4294967295
    kernel.shmmni = 4096
    kernel.sem = 250 32000 100 128
    net.ipv4.ip_local_port_range = 9000 65500
    net.core.rmem_default = 262144
    net.core.rmem_max = 4194304
    net.core.wmem_default = 262144
    net.core.wmem_max = 1048586
    ```

    输入下面的命令使下面的生效：

    ```
    [root@oracledb ~]# /sbin/sysctl -p
    ```

    在文件/etc/sysctl.conf中kernel.shmall和kernel.shmmax是存在的，需要先注释掉。

6. 修改系统资源限制

    ```
    [root@oracledb ~]# vim /etc/security/limits.conf 在文件末尾添加如下几行
    oracle soft nproc 2047
    oracle hard nproc 16384
    oracle soft nofile 1024
    oracle hard nofile 65536
    oracle soft stack 10240
    ```

    ```
    [root@oracledb ~]# vim /etc/pam.d/login
    session required pam_namespace.so #在此行下面添加一条pam_limits.so
    session required pam_limits.so
    ```

7. 创建安装目录及设置权限

    ```
    [root@oracledb ~]# mkdir -p /opt/oracle/app
    [root@oracledb ~]# mkdir -p /opt/oracle/oradata
    [root@oracledb ~]# chmod 755 /opt/oracle
    [root@oracledb ~]# chmod 775 /opt/oracle/app
    [root@oracledb ~]# chown oracle.oinstall -R /opt/oracle
    ```

8. 设置oracle用户环境变量，注意需要切换到oracle用户

    ```
    [oracle@oracledb ~]# vim ~/.bash_profile 在文件末尾添加下面几行
    export ORACLE_BASE=/opt/oracle/app
    export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
    export PATH=$PATH:$ORACLE_HOME/bin
    export ORACLE_SID=orcl
    ```

    输入下面命令使.bash_profile文件的设置立即生效
    
    ```
    [root@oracledb ~]# source .bash_profile
    ```

9. 关闭SELINUX

    ```
    [root@oracledb ~]# sed -i "s/SELINUX=enforcing/SELINUX=disabled/" /etc/selinux/config
    [root@oracledb ~]# setforce 0
    ```

10. 关闭防火墙
    
    ```
    [root@oracledb ~]# service iptables stop
    [root@oracledb ~]# chkconfig iptables off
    ```

11. 设置FTP

    ```
    安装vsftpd:
    [root@oracledb ~]# yum -y install vsftpd
    启动vsftpd:
    [root@oracledb ~]# service vsftpd start
    配置vsftpd:
    [root@oracledb ~]# vim /etc/vsftpd/vsftpd.conf 修改下面三项配置
    chroot_local_user=YES
    chroot_list_enable=YES
    chroot_list_file=/etc/vsftpd/chroot_list
    将oracle用户添加到chroot_list文件中
    [root@oracledb ~]# vim /etc/vsftpd/chroot_list
    oracle
    修改完成配置，重启vsftpd:
    [root@oracledb ~]# service vsftpd restart
    ```

###静默安装Oracle

1. 上传Oracle安装包

    利用SSH Secure Shell将安装文件linux.x64_11gR2_database_1of2.zip、linux.x64_11gR2_database_2of2.zip上传至oracle家目录/opt/oracle

2. 利用oracle用户登录并解压安装包

    ```
    [oracle@oracledb ~]# unzip linux.x64_11gR2_database_1of2.zip
    [oracle@oracledb ~]# unzip linux.x64_11gR2_database_2of2.zip
    ```

3. 修改Response File: db_install.rsp

    ```
    [oracle@oracledb ~]# cp /opt/oracle/database/response/db_install.rsp /opt/oracle/db_install.rsp
    [oracle@oracledb ~]# vim /opt/oracle/db_install.rsp 
    ```

    下面列出db_install.rsp中需要修改的项：

    ```
    oracle.install.option=INSTALL_DB_SWONLY
    ORACLE_HOSTNAME=oracledb
    UNIX_GROUP_NAME=oinstall
    INVENTORY_LOCATION=/opt/oracle/oraInventory
    SELECTED_LANGUAGES=en,zh_CN
    ORACLE_HOME=/opt/oracle/app/product/11.2.0/dbhome_1
    ORACLE_BASE=/opt/oracle/app
    oracle.install.db.InstallEdition=EE
    oracle.install.db.isCustomInstall=true
    oracle.install.db.DBA_GROUP=dba
    oracle.install.db.OPER_GROUP=oinstall
    oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
    oracle.install.db.config.starterdb.globalDBName=orcl
    oracle.install.db.config.starterdb.SID=orcl
    oracle.install.db.config.starterdb.characterSet=AL32UTF8
    oracle.install.db.config.starterdb.memoryOption=true
    oracle.install.db.config.starterdb.memoryLimit=512
    oracle.install.db.config.starterdb.password.ALL=oracle
    oracle.install.db.config.starterdb.control=DB_CONTROL
    oracle.install.db.config.starterdb.storageType=FILE_SYSTEM_STORAGE
    oracle.install.db.config.starterdb.fileSystemStorage.dataLocation=/opt/oracle/oradata
    DECLINE_SECURITY_UPDATES=true
    ```

    修改db_install.rsp文件执行权限：

    ```
    [oracle@oracledb ~]# chmod 777 /opt/oracle/db_install.rsp
    ```

4. 静默安装Oracle数据库

    ```
    [oracle@oracledb ~]# /opt/oracle/database/runInstaller -silent -responseFile /opt/oracle/db_install.rsp
    ```

    Response File文件的路径要写绝对路径，不要写相对路径。

    安装过程中可以使用tailf命令，查看安装日志文件，安装日志文件的路径和文件名都会在安装窗口中打印出来的。

###静默配置监听

* 静默配置监听

    ```
    [oracle@oracledb ~]# cp /opt/oracle/database/response/netca.rsp /opt/oracle/net_create.rsp
    [oracle@oracledb ~]# chmod 777 /opt/oracle/net_create.rsp
    [oracle@oracledb ~]# /opt/oracle/app/product/11.2.0/dbhome_1/bin/netca /silent /responsefile /opt/oracle/net_create.rsp
    ```

###静默创建数据库
1. 修改创建数据库的响应文件

    ```
    [oracle@oracledb ~]# vim /opt/oracle/db_create.rsp
    ```

    db_create.rsp文件内容如下：

    ```
    [GENERAL]
    RESPONSEFILE_VERSION = "11.2.0"
    OPERATION_TYPE = "createDatabase"
    [CREATEDATABASE]
    GDBNAME = "orcl.LK"
    TEMPLATENAME = "General_Purpose.dbc"
    CHARACTERSET = "AL32UTF8"
    TOTALMEMORY = "1024"
    ```

2. 静默创建数据库

    ```
    [oracle@oracledb ~]# /opt/oracle/app/product/11.2.0/dbhome_1/bin/dbca -silent -responseFile /opt/oracle/db_create.rsp
    ``` 
    等待一段时间后，界面上的东西都会消失掉，而且不出现输入密码的提示，不知道为什么会这样，但我还是输入两次密码，然后过一会就开始显示创建进度，我设置的密码为oracle。

###重要文件
* /opt/oracle/app/product/11.2.0/dbhome_1/dbs/init.ora
* /opt/oracle/app/product/11.2.0/dbhome_1/dbs/spfileorcl.ora
* pfile和sfile到底是什么现在还是不明白，但是他们是导致服务起不来的一个重要因素

###验证
1. 登录Oracle

    ```
    sqlplus / as sysdba
    ```

2. 启动服务
    ![startup](/images/2015-10-16-Monitor-Oracle/startup.jpg)

3. 利用system用户登录，查询v$tablespace的name字段
    ![name](/images/2015-10-16-Monitor-Oracle/name.jpg)

4. 启动1521端口监听

    ```
    lsnrctl start
    ```

    ![listen](/images/2015-10-16-Monitor-Oracle/listen.jpg)

    停止监听：lsnrctl stop
    
    查看监听状态：lsnrctl status

##使用一体化监控系统监控Oracle

现在还是没监控起来，报错如下图：
![MonitorError](/images/2015-10-16-Monitor-Oracle/MonitorError.jpg)

后来把代理接口改成2.53，并把JAVA_HOME的位置指为2.53上jdk的位置，就可以正确获取到数据，这样并没有完全的解决问题，首先是为什么2.93上自己的agent获取不到Oracle相关的监控数据，再就是2.93属于云平台内部的云主机，是不允许外面2.53这样的机器来提取数据的。

最后问题解决了，我们的一体化监控系统有iRadar Server和iRadar Agent，版本在不断的变化，当时出现监控获取不到数据就是因为2.93上的agent没有用最新的。
