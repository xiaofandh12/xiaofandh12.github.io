---
layout: post
title: OpenStack Ceilometer新增节点内存监控项
description: OpenStack, Ceilometer, Metering
category: blog
---

##文档
* [Github Ceilometer](https://github.com/openstack/ceilometer)
* [Launchpad Ceilometer](https://launchpad.net/cilometer)
* [Ceilometer Developer Doc](http://docs.openstack.org/developer/ceilometer/)
* [Ceilometer Administrator Guide Doc](http://docs.openstack.org/admin-guide-cloud/telemetry.html)
* [Ceilometer Config Ref](http://docs.openstack.org/kilo/config-reference/content/ch_configuring-openstack-telemetry.html)
* [Ceilometer API Ref](http://developer.openstack.org/api-ref-telemetry-v2.html)
* [孔令贤-关于python中的setup.py](http://lingxiankong.github.io/blog/2013/12/23/python-setup/)
* [SetupTools Doc](http://pythonhosted.org/setuptools/)
* [pbr Doc](http://docs.openstack.org/developer/pbr/)
* [oslo Doc](https://wiki.openstack.org/wiki/Oslo)
* [oslo vmware](http://rpmfind.net/linux/RPM/fedora/21/x86_64/p/python-oslo-vmware-0.3-2.fc21.noarch.html)

##几个有用的目录
* Ceilometer配置文件所在目录：/etc/ceilometer/
* Ceilometer日志文件所在目录：/var/log/ceilometer/

##本次新增监控项需要操作的目录或文件
假设当前已经在Ceilometer目录下了，那么需要操作的目录或文件为:

* ./ceilometer/hardware/pollsters/
* ./ceilometer/hardware/inspector/base.py
* ./ceilometer/hardware/inspector/snmp.py
* ./setup.cfg
* ./etc/ceilometer/pipeline.yaml

##新增节点内存监控项的流程
本小结内容完全参考于《在Ceilometer中新增监控插件-刘从祥》

1. 在./ceilometer/hardware/pollsters/下增加os.py,定义节点内存监控项的poller:systemMemoryTotalPollster。os.py文件内容为：
    
    ```
    from ceilometer.hardware import plugin
    from ceilometer.hardware.pollsters import util
    from ceilometer import sample

    class _Base(plugin.HardwarePollster):
        CACHE_KEY = 'system'
        INSPECT_METHOD = 'inspect_system'

    class systemMemoryTotalPollster(_Base):
        def generate_one_sample(host, c_data):
            (system, info) = c_data
            return util.make_sample_from_host(host, name='system.mem_total', type=sample.TYPE_GAUGE, unit='GB', volume=info.mem_total, res_metadata=system)
    ```

2. 新增通过SNMP协议获取数据的实现

    修改./ceilometer/hardware/inspector/base.py，在适当的位置增添下面两行代码：
    
    ```
    System = collections.namedtuple('System', ['device', 'path'])
    SystemStats = collections.namedtuple('SystemStats', ['mem_total'])
    ```

    修改./ceilometer/hardware/inspector/snmp.py，在类SNMPInspector(base.Inspector)中新增数据成员，增加相关OID的定义：
    
    ```
    _system_name_oid = '1.3.6.1.2.1.1.5.0'
    _system_memory_total_oid = '1.3.6.1.2.1.25.2.2.0'
    ```

    修改./ceilometer/hardware/inspector/snmp.py，在类SNMPInspector(base.Inspector)中新增获取数据的函数：inspect_system(self, host)，该函数的名称应该与os.py中的类_Base(plugin.HardwarePollster)中的INSPECT_METHOD的值一样。
    
    ```
    def inspect_system(self, host):
        memory_total = long(0)

        host_name = ''
        try:
            system_memory_total = self._get_value_from_oid(self._system_memory_total_oid, host)
            host_name = self._get_value_from_oid(self._system_name_oid, host)

            memory_total = float(system_memory_total) / float(1024 * 1024)
            memory_total = float('%.2f' % memory_total)
        except:
            pass


        System = base.System(device=str(host_name), path=str(host_name))
        SystemStats = base.SystemStats(mem_total=long(memory_total))

        yield (System, SystemStats)
    ```


3. 修改配置文件
    
    修改配置文件./etc/ceilometer/pipeline.yaml
    
    ```
    -name: hardware_system_source
     interval: 600
     meters:
        - "hardware.system.*"
     resources:
        - snmp://172.31.2.52
        - snmp://172.31.2.50
        - snmp://172.31.2.55
        - snmp://172.31.2.54
        - snmo://172.31.2.51
     sinks:
        - meter_sink
    ```

    修改配置文件./setup.cfg，在ceilometer.poll.central=下面新增插件的配置，meter和poller的名称与之前文件中的定义要一致
    
    ```
    hardware.system.mem_total = ceilometer.hardware.pollsters.os:systemMemoryTotalPollster
    ```

4. 验证
    
    重新编译安装ceilometer，在ceilometer源码目录下执行下面的命令：
    
    ```
    python setup.py build
    python setup.py install
    ```

    检查/etc/ceilometer/pipeline.yaml是否满足要求

    重启相关服务，如重启openstack-ceilometer-api服务的命令为：
    
    ```
    service openstack-ceilometer-api restart
    ```

    使用下面的命令可以查看openstack相关服务：

    ```
    service --status-all | grep openstack
    ```

##新增监控项时遇到的问题
1. 运行命令：python setup.py install出现下面的错误：
![problem1](/images/2015-10-10-Add-Monitoring/problem1.jpg)

    从pkg_resource.VersionConflict:(pbr 0.8.0 (/usr/lib/python2.6/site-packages), Requirement.parse('pbr>=1.3'))可以看出是pbr的版本有问题。本机中的pbr的版本为0.8.0。

    setup.py中的代码非常简单，主要就是在调用pbr。关于setup.py、pbr等，可以参看博客：[python-setup](http://lingxiankong.github.io/blog/2013/12/23/python-setup/)。

    在requirements.txt中有对需要安装的各种模块的版本的要求列表，如下图。从中可以看出对pbr的要求是pbr>=0.6, <1.0，我们机器中的pbr版本满足要求。
    ![requirements](/images/2015-10-10-Add-Monitoring/requirements.jpg)

    command python setup.py egg_info failed with code 1 in /tmp/pip-build-root/oslo.vmware storing complete log in /root/.pip/pip.log,从这句话中可以看出主要是在安装oslo.vmware时出现了错误，猜测是安装oslo.vmware需要pbr的版本高于1.3。还可以看到完整的日志是存于/root/.pip/pip.log中，查看该日志可以看到确实是在安装oslo.vmware时出现了错误。下面是/root/.pip/pip.log中相关内容的截图：
    ![log](/images/2015-10-10-Add-Monitoring/log.jpg)

    如何解决：修改ceilometer-2014-1-1/requirements.txt，将oslo.vmware>=0.2注释掉。

2. Meter类型不能为非数值型数据

   在文档[New Measurements](http://docs.openstack.org/developer/cilometer/new_meters.html)中，可以看到新增的Meter类型有三种：Cumulative, Gauge, Delta。

   在文档[Events](http://docs.openstack.org/admin-guide-cloud/telemetry-events.html)中有写到：
   > While a sample represents a single, numeric datapoint within a time-series, an event is a broader concept that represents the state of a resource at a point in time. The state may be described using various data types including non-numeric data such as an instance's flavor. In general, events represent any action made in the OpenStack system.
   
   可以知道，Meter只能用来收集数值型的数据。

   关于这个问题我也在邮件列表opstack-dev中问了，下面的截图是Igor Degtiarov的回复：
   ![CeilometerMeterType](/images/2015-10-10-Add-Monitoring/CeilometerMeterType.jpg)

   关于Openstack的邮件列表和IRC的使用问题，我会再另写一篇博客来解释。
