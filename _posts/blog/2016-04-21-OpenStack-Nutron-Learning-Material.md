---
layout: post
title: OpenStack Neutron -- 学习资料
description: OpenStack, Nutron, Learning Material
category: blog
---

## 学习什么

* neutron代码的整体架构，消息通知、rpc如何实现，RESTful API如何实现
* neutron的部署，常见问题的定位方法
* neutron的配置文件
* neutron的数据库设计，数据库中各表格的作用及其关联关系
* neutron-server的启动流程及其作用
* neutron-rpc-server的启动流程及其作用
* neutron-openvswitch-agent的启动流程及其作用
* neutron-dhcp-agent的启动流程及其作用
* neutron-l3-agent的启动流程及其作用
* neutron-linuxbridge-agent的启动流程及其作用
* openvswitch、openflow、linuxbridge、iptables，tap device, veth pair的原理及其作用
* plugin, driver, agent的关联关系，及作用
* flat, vlan, gre, vxlan的网络模式是如何实现的
* 如何与keystone交互进行身份认证，policy.json的原理和作用
* nova会调用哪些neutron的API，流程是怎样的
* neutron处理API请求的流程
* firewall as a service, load banalance as a service, vpn as a service， security group
* neutron的HA如何实现
* neutron各种部署方式下，两个虚拟机之间如何通信以及虚拟机如何与外网通信
* 关注邮件列表、IRC、OpenStack Summit，了解neutron最新动态
* SDN/NFV


## 源码
[OpenStack Neutron](https://github.com/openstack/neutron)

* [OpenStack之Neutron入门一](http://www.aboutyun.com/forum.php?mod=viewthread&tid=9523&highlight=Neutron%2B%2B%C8%EB%C3%C5)
* [OpenStack之Neutron入门二](http://www.aboutyun.com/thread-9517-1-1.html)
* [OpenStack之Neutron入门三](http://www.aboutyun.com/thread-9537-1-1.html)
* [openstack源码分析 - Paste Deployment介绍 ](http://blog.chinaunix.net/uid-20940095-id-4105407.html)
* [openstack neutron分析（1）——neutron-server启动过程分析 ](http://www.aboutyun.com/thread-9527-1-1.html)
* [openstack Neutron分析（2）—— neutron-l3-agent](http://www.aboutyun.com/thread-9529-1-1.html)
* [openstack Neutron分析（3）—— neutron-dhcp-agent源码分析](http://www.aboutyun.com/thread-9533-1-1.html)
* [openstack Neutron分析（4）—— neutron-l3-agent中的iptables ](http://www.aboutyun.com/thread-9536-1-1.html)
* [openstack Neutron分析（5）-- neutron openvswitch agent ](http://www.aboutyun.com/thread-9538-1-1.html)
* [openstack nova 源码分析1-setup脚本 ](http://www.aboutyun.com/thread-10090-1-1.html)
* [openstack nova 源码分析2之nova-api,nova-compute](http://www.aboutyun.com/thread-10091-1-1.html)
* [openstack nova 源码分析3-nova目录下的service.py、driver.py](http://www.aboutyun.com/thread-10092-1-1.html)
* [openstack nova 源码分析4-1 -nova/virt/libvirt目录下的connection.py ](http://www.aboutyun.com/thread-10094-1-1.html)
* [openstack nova 源码分析4-2 -nova/virt/libvirt目录下的connection.py ](http://www.aboutyun.com/thread-10095-1-1.html)
* [OpenStack Nova源码结构解析 ](http://www.aboutyun.com/thread-10105-1-1.html)
* [ OpenStack源码探秘（一）——Nova-Scheduler](http://blog.csdn.net/networm3/article/details/8783667?utm_source=tuicool&utm_medium=referral)
* [别以为真懂Openstack: 虚拟机创建的50个步骤和100个知识点(1)](http://www.sxt.cn/u/756/blog/2780)
* [别以为真懂Openstack: 虚拟机创建的50个步骤和100个知识点(2)](http://www.sxt.cn/u/756/blog/2778)
* [别以为真懂Openstack: 虚拟机创建的50个步骤和100个知识点(3)](http://www.sxt.cn/u/756/blog/2797)
* [别以为真懂Openstack: 虚拟机创建的50个步骤和100个知识点(4)](http://www.sxt.cn/u/756/blog/2798)
* [别以为真懂Openstack: 虚拟机创建的50个步骤和100个知识点(5)](http://blog.csdn.net/xiangpingli/article/details/47857123)

## 文档
### [OpenStack Doc官网](http://docs.openstack.org)

* [OpenStack Installation Guide for Red Hat Enterprise Linux and CentOS](http://docs.openstack.org/mitaka/install-guide-rdo/)
* [OpenStack Networking Guide](http://docs.openstack.org/mitaka/networking-guide/)
* [OpenStack Operations Guide](http://docs.openstack.org/openstack-ops/content/)
	* [Architecture -> Chapter 7. Network Desing](http://docs.openstack.org/openstack-ops/content/network_design.html)
	* [Operations -> Chapter 12. Network Troubleshooting](http://docs.openstack.org/openstack-ops/content/network_troubleshooting.html)
* [OpenStack Administrator Guide](http://docs.openstack.org/admin-guide/)
	* [Networking](http://docs.openstack.org/admin-guide/networking.html)
* [OpenStack High Availability Guide](http://docs.openstack.org/ha-guide/)
	* [OpenStack network nodes](http://docs.openstack.org/ha-guide/networking-ha.html)
* [OpenStack Security Guide](http://docs.openstack.org/security-guide/)
	* [Networking](http://docs.openstack.org/security-guide/networking.html)
* [OpenStack Architecture Design Guide](http://docs.openstack.org/arch-design/)
	* [Network focused](http://docs.openstack.org/arch-design/network-focus.html)
* [OpenStack API Complete Reference](http://developer.openstack.org/api-ref.html)
	* [Networking API v2.0 (CURRENT)](http://developer.openstack.org/api-ref-networking-v2.html)
	* [Networking API v2.0 extensions (CURRENT)](http://developer.openstack.org/api-ref-networking-v2-ext.html)
* [OpenStack End User Guide](http://docs.openstack.org/user-guide/)
	* [OpenStack command-line clients -> Create and manage networks](http://docs.openstack.org/user-guide/cli_create_and_manage_networks.html)
	* [OpenStack Python SDK -> Networking](http://docs.openstack.org/user-guide/sdk_neutron_apis.html)
* [OpenStack Command-Line Interface Reference](http://docs.openstack.org/cli-reference/)
	* [Networking service command-line client](http://docs.openstack.org/cli-reference/neutron.html)
	* [Networking miscellaneous command-line client](http://docs.openstack.org/cli-reference/neutron-misc.html)
* [Open source software for application development](http://developer.openstack.org)
* [OpenStack Developer Documentation](http://docs.openstack.org/developer/openstack-projects.html)
	* [Networking service Developer Documentation(neutron)](http://docs.openstack.org/developer/neutron/)
	* [BGP-MPLS VPN Networking service Plug-in(networking-bgpvpn)](http://docs.openstack.org/developer/networking-bgpvpn/)
	* [Calico Networking service Plug-in(networking-calico)](http://docs.openstack.org/developer/networking-calico/)
	* [L2GW Netwoking service Plug-in (networking-l2gw)](http://docs.openstack.org/developer/networking-l2gw/)
	* [MidotNet Networking service Plug-in(networking-midonet)](http://docs.openstack.org/developer/networking-midonet/)
	* [OVN Networking service Plug-in(networking-ovn)](http://docs.openstack.org/developer/networking-ovn/)
	* [PowerVM Networking service Plug-in(networking-powervm)](http://docs.openstack.org/developer/networking-powervm/)
	* [Service Function Chaining Networking service Plug-in(networking-sfc)](http://docs.openstack.org/developer/networking-sfc/)

### [RDO Doc](https://www.rdoproject.org/documentation/)

* [trystack](http://trystack.org)
* [Networking](https://www.rdoproject.org/documentation/networking/)
	* [OpenStack Networking in Too Much Detail](https://www.rdoproject.org/networking/networking-in-too-much-detail/)
	* [Using GRE Tenant Networks](https://www.rdoproject.org/networking/using-gre-tenant-networks/)
    * [Difference between Floating IP and private IP](https://www.rdoproject.org/networking/difference-between-floating-ip-and-private-ip/)
    
### [Product Documentation for Red Hat OpenStack Platform](https://access.redhat.com/documentation/en/red-hat-openstack-platform/)

### [Mirantis Resources](https://www.mirantis.com/openstack-resources/)

* [Mirantis OpenStack 7.0 NFVI Deployment Guide](https://content.mirantis.com/MOS-7-NFVI-Whitepaper-Landing-Page.html)
		
### [Rakspace doc](http://docs.rackspace.com)
    
    * [Neutron Networking: Neutron Routers and the L3 Agent](https://developer.rackspace.com/blog/neutron-networking-l3-agent/)

### SDN/NFV

* [Open vSwitch](http://openvswitch.org)

* [Open vSwitch Documentation](http://openvswitch.org/support/)

* [OpenFlow](http://archive.openflow.org)

* [OpenFlow Documents](http://archive.openflow.org/wp/documents/)

* [OPEN NETWORKING FOUNDATION](https://www.opennetworking.org/)

* [OPEN DAYLIGHT](https://www.opendaylight.org)

* [opnfv](https://www.opnfv.org)

* [Neutron-Service Function Chaining](http://docs.openstack.org/developer/networking-sfc)

* [vmware NSX](http://www.vmware.com/cn/products/nsx)

* [sdnlab](http://www.sdnlab.com)

* [Dragonflow](http://wiki.openstack.org/wiki/Dragonflow)

* [tacker](http://docs.openstack.org/developer/tacker)

## 书籍
* 《TCP/IP详解 卷一：协议》、《TCP/IP详解 卷二：实现》 by W.Richard Stenvens
* 《UNIX网络编程 卷一：套接字联网API》、《UNIX网络编程 卷二：进程间通信》 by W.Richard Stevens
* 《UNIX环境高级编程》by W.Richard Stevens
* 《Computer Networks (Fifth Edition)》by Andrew S. Tanenbaum
* 《Computer Systems A Programmmer's Perspective (Secone Edition)》 by Randal E. Bryant
* 《Learning OpenStack Networking (Neutron)》
* 《OpenStack设计与实现》
* 《OpenStack开源云王者归来》
* 《云计算与OpenStack 虚拟机Nova篇》

## 博客
* [有云博客](https://www.ustack.com/about/blog/)
* [Mirants Blog](https://www.mirantis.com/blog/)
* [RDO Community Blogs](http://techs.enovance.com)
* [RACKSPACE DEVELOPER BLOG](https://developer.rackspace.com/blog/)
* [孔令贤](http://lingxiankong.github.io)
* [Blogs - Red Hat Customer Portal](https://access.redhat.com/blogs/)
* [开发人员必读openstack网络基础1:什么是L2、L3](http://www.aboutyun.com/forum.php?mod=viewthread&tid=9647&highlight=%BF%AA%B7%A2%C8%CB)
* 一系列文章[Neutron 理解(1)：Neutron所实现的虚拟化网络(How Neutron Virtualizes Network)](http://www.cnblogs.com/sammyliu/p/4622563.html)
* [openstack(juno) 入门(总结篇)二十七:openstack排除故障及常见问题记录](http://www.aboutyun.com/thread-11718-1-1.html)
* [Neutron理解(9):How Nova Implements Security Group and How Neutron Implements Virtual Firewall](http://www.cnblogs.com/sammyliu/p/4675991.html)
* [OpenStack Neutron FWaaS学习(by quqi99)](http://blog.csdn.net/quqi99/article/details/9163469)
* [OpenStack中的防火墙(by quqi99)](http://blog.csdn.net/quqi99/article/details/7447233)
* [初探Openstack Neutron DVR](http://www.sxt.cn/u/756/blog/3168)
* [openstack的虚拟机网卡、网桥等（tap、qbr、qvb、qvo）mtu设置](http://my.oschina.net/tantexian/blog/648973?fromerr=DOzA8FMf)
* [云计算：openstack neutron(tap、qvb、qvo、qbr详解) ](http://blog.chinaunix.net/uid-7374279-id-5096216.html)
* [neutron的基本原理](http://www.cnblogs.com/popsuper1982/p/3800233.html)
* [OpenvSwitch完全使用手册](http://my.oschina.net/tantexian/blog/648965?fromerr=ZkIfFQ0j)
* [专注云计算-学习OpenStack](http://www.cnblogs.com/sammyliu/category/636967.html)
* [技术点详解---IPSec VPN基本原理](http://www.cnblogs.com/phoenixzq/archive/2013/05/18/3085366.html)
* [NAT技术介绍(转)](http://my.oschina.net/tantexian/blog/648957?fromerr=qrqJiZS3)
* [IPsec技术介绍(转)](http://my.oschina.net/tantexian/blog/648955?fromerr=pN2o73uW)
* [IPSEC与SSL/TLS的比较（转）](http://my.oschina.net/tantexian/blog/648954?fromerr=VtjD9AqD)
* [IPSec VPN与SSL VPN 比较（转）](http://my.oschina.net/tantexian/blog/648956?fromerr=1xXbamre)
* [项目经理 VS 产品经理 （工作职责和要求）](http://my.oschina.net/tantexian/blog/648907?fromerr=v7EQePQK)
* [OpenStack开发过程中常用Git操作场景（转）](http://my.oschina.net/tantexian/blog/645113?fromerr=tyJoT3Vo)
* [centos 7.0 网卡配置及重命名教程（转）](http://my.oschina.net/tantexian/blog/634534?fromerr=y0mtbX9s)
* [CentOS 7.0，启用iptables防火墙(转)](http://my.oschina.net/tantexian/blog/634533?fromerr=CdMKx3rA)
* [centos7 VMware workstation 10 添加多网卡及重命名为ethx（eth0，eth1失败）(还想再添加网卡eth1???)
](http://my.oschina.net/tantexian/blog/634532?fromerr=8sIGSqmD)
* [vmware centos7 clone mac地址导致 Failed to start LSB: Bring up/down networking.](http://my.oschina.net/tantexian/blog/634529?fromerr=smfHcI90)
* [centos7将网卡加入到网桥后， Missing config file br-ex，网卡无法正常启动问题解决](http://my.oschina.net/tantexian/blog/634528?fromerr=TISUHsxt)
* [OpenStack中的LoadBalancer(负载均衡)功能使用实例](http://www.cnblogs.com/biangbiang/archive/2013/05/29/3105900.html)
* [OpenStack Neutron之Load Balance](http://www.ibm.com/developerworks/cn/cloud/library/1506_xiaohh_openstacklb/)
* [负载均衡之Haproxy配置详解（及httpd配置）](http://my.oschina.net/tantexian/blog/626174?fromerr=mQOuYt6U)
* [系统原理分析架构-二-CDN内容分发网络](http://my.oschina.net/tantexian/blog/626173?fromerr=7CmzQSa5)
* [系统原理分析架构-一-DNS负载均衡](http://my.oschina.net/tantexian/blog/626171?fromerr=zgUMAoCy)
* [系统原理分析架构-六-负载均衡（定义及介绍及LVS/Nginx/Haproxy比较）](http://my.oschina.net/tantexian/blog/626170?fromerr=2D6S9DyZ)
* [深入理解openstack网络架构（1） ](http://blog.csdn.net/halcyonbaby/article/details/41524447/)
* [深入理解openstack网络架构（2） ---- Basic Use Cases](http://blog.csdn.net/halcyonbaby/article/details/41578293/)
* [深入理解openstack网络架构（3） ---- 路由](http://blog.csdn.net/halcyonbaby/article/details/41604459)
* [深入理解openstack网络架构（4） ---- 连接到publi network](http://blog.csdn.net/halcyonbaby/article/details/41628891)
* [网络虚拟化技术（一）：Linux网络虚拟化](http://blog.csdn.net/halcyonbaby/article/details/411369151)
* [网络虚拟化技术（二）：TUN/TAP MACVLAN MACVTAP](http://blog.csdn.net/halcyonbaby/article/details/41269225)
* [Open vSwitch工作原理](https://blog.kghost.info/2014/11/19/openvswitch-internal/)
* [OpenStack neutron floating ips与iptables深入分析](http://blog.csdn.net/starean/article/details/16860819)
## Mailing List and IRC
* openstack-dev@lists.openstack.org
* /#openstack-neutron, /#openstack-meeting-3
