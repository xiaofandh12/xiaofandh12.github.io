---
layout: post
title: Ceilometer的API调用流程分析
description: OpenStack, Ceilometer, Meter List, API
category: blog
---

## 文档

* [WSGI是什么](http://segmentfault.com/a/1190000003069785)
* [通过demo学习OpenStack开发所需的基础知识--API服务(1)](http://segmentfault.com/a/1190000003718598)
* [通过demo学习OpenStack开发所需的基础知识--API服务(2)](http://segmentfault.com/a/1190000003718606)
* [通过demo学习OpenStack开发所需的基础知识--API服务(3)](http://segmentfault.com/a/1190000003810294)
* [Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.html)
* [ceilometer api源码](http://www.cnblogs.com/yippee/p/4699927.html)
* [浅谈Python装饰器](http://blog.csdn.net/mdl13412/article/details/22608283)
* [PyMongon Documentation Tutorial](https://api.mongodb.org/python/current/tutorial.html)
* [Pecan Documentation](http://pecan.readthedocs.org/en/latest/index.html)
* [WSME Documentation](http://wsme.readthedocs.org/en/latest/index.html)
* [argparse](https://docs.python.org/2/library/argparse.html?highlight=argparse#action)
* [oslo](https://wiki.openstack.org/wiki/Oslo)
* [oslo.config](http://docs.openstack.org/developer/oslo.config)
* [python built-in function](https://docs.python.org/2.7/library/functions.html)
* [stevedore](http://docs.openstack.org/developer/stevedore/)
* [setuptools](https://pythonhosted.org/setuptools/)
* [pbr](http://docs.openstack.org/developer/pbr)
* [Telemetry API v2](http://developer.openstack.org/api-ref-telemetry-v2.html)
* [Routes Documentation](https://routes.readthedocs.org/en/latest/)
* [Paste Documentation](http://pythonpaste.org)
* [PasteDeploy Documentation](http://pythonpaste.org/deploy/)
* [WebOb](http://webob.org)

## 本文在讲什么

本文的讲解是基于目前ceilometer最新的Liberty版本

![Article About](/images/2015-11-12-Ceilometer-Meter-List-API/2015-11-12-Article-About.png)

在文章[Ceilometer数据采集原理](http://xiaofandh12.github.io/Ceilometer-Collect-Data/)中曾提到(这也是来自于官方文档的说法)三种从Ceilometer获取数据的方式:

* with the RESTful API
* with the command line interface
* with Metering tab on an openstack dashboard

从这三种方式的描述来看，我们知道从Ceilometer获取数据的方式归根结底还是通过RESTful API来实现的。

本文就尝试讲述一下，通过python-ceilometerclient提供的ceilometer命令来获取数据的全过程，也就是命令ceilometer --debug meter-list来获取监控项列表的过程，也即上图中蓝色部分。

## 基础知识

下面这些基础知识在上面文档部分都可以找到相应的文档作为参考，理解了这些基础知识对理解ceilometer API的调用流程非常有帮助，我对这些也是处于一知半解的状态:)

* pymongo --> 用来连接MongoDB的python包 
* oslo.config --> OpenStack通用库之一，用来解析命令行和配置文件中的配置选项
* pecan --> 具体实现WSGI的一个python框架
* wsme --> Web Service Made Easy，专门用于实现REST服务的typing库
* argparse --> python标准库中推荐用来编写命令行程序的工具，python-ceilometerclient使用的正是argparse
* stevedore --> 用于运行时动态载入代码的python包
* setuptools and pbr
* python decorator --> python装饰器相关知识，python内置装饰器:@staticmethod, @classmethod, @property
* python built-in function --> python内置函数相关知识

## 什么是RESTful API、WSGI、pecan

### RESTful API
REST的全称是Representational State Transfer(表征状态转移)，是Roy Fielding在他的博士论文[Architectural Styles and the Design of Network-based Software Architecture]()中提出的一种软件架构风格，而我们一般把满足这种设计风格的API称为RESTful API。

具体到使用Python来提供RESTful API时，又提出了一个WSGI的规范。

### WSGI

![WSGI](/images/2015-11-12-Ceilometer-Meter-List-API/2015-11-13-WSGI.png)

WSGI的全称是Web Server Gateway Interface(Web服务器网关接口)，是python语言中所定义的Web服务器和Web应用程序或框架之间的通用接口标准，它对应于Java中的Servelet。

下面是一些学习资源：

* [WSGI简介](http://blog.csdn.net/on_1y/article/details/18803563/)
* [PEP 3333](https://www.python.org/dev/peps/pep-3333/)
* [WSGI readthedocs](http://wsgi.readthedocs.org/en/latest)
* [WSGI参考实现](https://pypi.python.org/pypi/wsgiref)
* [WSGI源码阅读](https://github.com/minixalpha/SourceLearning/tree/master/wsgiref-0.1.2)
* [WSGI研究](http://blog.kenshinx.me/blog/wsgi-research)
* [WSGI初探](http://linluxiang.iteye.com/blog/799163)

### Pecan

在OpenStack的项目中实现RESTful API的Web框架主要有两种方式：

* Paste + PasteDeploy + Routes + WebOb
* Pecan

在OpenStack早期的项目中(Nova, Nutron, Keystone)都是使用的Paste + PasteDeploy + Routes + WebOb，这样的框架好处在于灵活性，但后来它的灵活性并没有抵消它的复杂性，于是在OpenStack后来的项目中也就不再使用这个框架了，但对于理解这些早期项目仍很有必要好好学习这种框架，尤其是这些早期项目都是OpenStack中最重要的一些项目。

Pecan是一个轻量级的Python的Web框架，OpenStack中的新项目全面的使用了此框架(如magnum)，Pecan还可以和PasteDeploy一起使用，Ceilometer就是如此。

## ceilometer --debug meter-list的调用流程分析

### 大概过程

* python-ceilometerclient
    
    > ceilometer --debug meter-list
    
    > curl GET http://172.31.2.51:8777/v2/meters

* Python Application的创建
* HTTP请求的解析
    
    > RootController
    
    > V2Controller
    
    > MetersController

* 实际去获取数据的两个函数
    
    > pecan.request.storage_conn.get_meters的执行
    
    > Meter.from_db_model的执行

### python-ceilometerclient

python-ceilometerclient为我们提供了ceilometer命令，通过该命令我们可以很方面的调用ceilometer提供的API。
在搭建好的环境中执行ceilometer --debug meter-list，我们除了可以得到数据库中meter列表外，还可以看到如下的curl命令：
    ```
    curl -i -X GET -H 'X-Auth-Token: 968b7f741482416899f477a8e3aafba7' -H 'Content-Type: application/json' -H 'Accept: application/json' -H 'User-Agent: python-ceilometerclient' http://172.31.2.51:8777/v2/meters
    ```
，在这条命令中比较关键的是：
    ```
    http://172.31.2.51:8777/v2/meters
    ```
，下面我们就分析ceilometer收到这个HTTP请求后是如何解析，如何使用Python Application去查询MongoDB数据库的

讲解过程会先给出一段文字解说，然后再给出相应的源码，看文字解说一定要阅读相应的源码方能更好理解。

### Python Application的创建过程
需要使用Python Application来接收HTTP请求，因此需要先创建Python Application。下面我们介绍使用Pecan+PasetDeploy创建Python Application的过程。

在配置文件api_pasete.ini中我们可以看到，PasteDeploy会调用ceilometer.api.app:app_factory。

* ../etc/ceilometer/api_paste.ini: [app:api-server]

    ```
    [pipeline:main]
    pipeline = request_id authtoken api-server

    [app:api-server]
    paste.app_factory = ceilometer.api.app:app_factory
    ```

下面我们再到../ceilometer/api/app.py中查看app_factory函数，可以看到app_factory函数返回的是一个VersionSelectorApplication类，VersionSelectorApplication是为了创建不同版本的Python Application，其中v1版本是不能用的，如果是v2版本则会去调用setup_app函数。

setup_app函数就是真正创建Python Application的函数，在此函数中最重要的就是调用了pecan.make_app函数，在此函数中最重要的就是指定了解析HTTP Request的RootController，是通过pecan_config.app.root参数指定的；另外一个重要的地方就是hoooks.DBHook，在这里面初始化了数据库的链接，关于这一点后面再做介绍。

* ../ceilometer/api/app.py: app_factory --> VersionSelectorApplication --> setup_app

    ```
    ...
    def setup_app(pecan_config=None, extra_hooks=None):
        app_hooks = [hooks.ConfigHook(),hooks.DBHook(),hooks.NotifierHook(),hooks.TranslationHook()]
        if extra_hooks:
            app_hooks.extend(extra_hooks)

        if not pecan_config:
            pecan_config = get_pecan_config()

        pecan.configuration.set_config(dict(pecan_config), overwrite=True)

        pecan_debug = CONF.api.pecan_debug
        if CONF.api.workers and CONF.api.works != 1 and pecan_debug:
            pecan_debug = False
            LOG.warning(_LW('pecan_debug cannot be enabled, if workers is > 1, the value is overrided with False'))

        app = pecan.make_app(
            pecan_config.app.root,
            debug=pecan_debug,
            force_cannonical=getattr(pecan_config.app, 'force_canonical',True)
            hooks=app_hooks,
            wrap_app=middleware.ParsableErrorMiddleware,
            guess_content_type_from_ext=False
        )

        return app
    ...
    class VersionSelectorApplication(object):
        def __init__(self):
            pc = get_pecan_config()

            def not_found(environ, start_response):
                start_response('404 Not Found', [])
                return []

            self.v1 = not_found
            self.v2 = setup_app(pecan_config=pc)

        def __call__(self, environ, start_response):
            if environ['PATH_INFO'].startswith('/v1/'):
                return self.v1(environ, start_response)
            return self.v2(environ, start_response)
    ...
    def app_factory(global_config, **local_conf):
        return VersionSelectorApplication()
    ```

在上面我们提到解析HTTP请求的RootController是通过pecan_config.app.root参数指定的，阅读代码可知pecan_config.app.root就是config.py中指定的，也就是ceilometer.api.controllers.root.RootController。这样Python Application就创建完成了，并且指定好了解析HTTP Request的RootController，也就是说/v2/meters/从RootController开始处理。

* ../ceilometer/api/config.py: app

    ```
    server = {
        'port': '8777',
        'host': '0.0.0.0'
    }
    app = {
        'root': 'ceilometer.api.controllers.root.RootController',
        'modules': ['ceilometer.api'],
    }
    ```

### HTTP请求的解析过程

在RootController我们可以看到它先创建了一个类属性:v2，于是所有的以/v2开头的HTTP Request都会由V2Controller来处理，/v2/meters/当然也不例外。

* ../ceilometer/api/controllers/root.py: RootController

    ```
    ...
    class RootController(object):
        def __init__(self):
            self.v2 = v2.V2Controller()
        def index(self):
            base_url = pecan.request.application_url
            available = [{'tag': 'v2', 'date': '2013-02-13T00:00:00Z',}]
            collected = [version_descriptor(base_url, v['tag'], v['date']) for v in available]
            versions = {'versions': {'values': collected}}
            return versions
    ```

下面我们再去看V2Controller，我们可以看到它有类属性:event_types,events,capabilities，也就是说/v2/event_types会由EventTypesController来处理，/v2/events/会由EventsController来处理，/v2/capabilities/会由CapabilitiesController来处理，那/v2/meters/呢？往下看，在```_lookup```函数中我们可以看到/v2/meters/会由MetersController来处理，关于```__lookup```可以参看官方文档。

* ../ceilometer/api/controllers/v2/root.py: V2Controller

    ```
    ...
    class V2Controller(object):
        event_types = events.EventTypesController()
        events = events.EventsController()
        capabilities = capabilities.CapabilitiesController()

        def __init__(self):
            self._gnocchi_is_enabled = None
            self._aodh_is_enabled = None
            self._aodh_url = None
        ...
        def __lookup(self, kind, *remainder):
            if (kind in ['meters', 'resources', 'samples'] and self.gnocchi_is_enabled):
                gnocchi_abort()
            elif kind == 'meters':
                return meters.MetersController(), remainder
            elif kind == 'resources':
                return resources.ResourcesController(), remainder
            elif kind == 'samples':
                return samples.SamplesController(), remainder
            elif kind == 'query':
                return QueryController(gnocchi_is_enabled=self.gnocchi_is_enabled, aodh_url=self.aodh_url,), remainder
            elif kind == 'alarms' and self.aodu_url:
                aodh_redirect(self.aodh_url)
            elif kind == 'alarms':
                return alarms.AlarmsController(), reaminder
            else:
                pecan.abort(404)
    ```

终于到了MetersController，这里就是最终处理HTTP请求:/v2/meters/的地方，貌似要结束了的样子。在这里最重要的就是最后那一行代码：``` return [Meter.from_db_model(m) for m in pecan.request.storage_conn.get_meters(limit=limit,**kwargs)] ```，在这行代码中有两个重要的函数调用：``` pecan.request.storage_conn.get_meters``` 以及 ``` Meter.from_db_model```。所以还没结束，下面还得分析这两个函数是如何执行的。

* ../ceilometer/api/controllers/v2/meters.py: MetersController

    ```
    ...
    class MetersController(rest.RestController):
        @pecan.expose()
        def __lookup(self, meter_name, *remainder):
            return MeterController(meter_name), remainder

        @wsme_pecan.wsexpose([Meter], [base.Query], int)
        def get_all(self, q=None, limit=None):
            rbac.enforce('get_meters', pecan.request)

            q = q or []

            limit = v2_utils.enforce_limit(limit)
            kwargs = v2_utils.query_to_kwargs(q, pecan.request.storage_conn.get_meters,['limit'],allow_timestamps=False)
            return [Meter.from_db_model(m) for m in pecan.request.storage_conn.get_meters(limit=limit,**kwargs)]
    ```

### pecan.request.storage_conn.get_meters的执行过程

要分析pecan.request.storage_conn.get_meters(limit=limit, **kwargs)的执行过程我们分两步：一是storage_conn是如何获得的，二是get_meters是如何执行的
。

#### stroage_conn的获得过程

storage_conn指的是数据库的链接，在```DBHook.__init__```中可以看出它是通过调用函数get_connection来获得的，而函数get_connection又调用了函数storage.get_connection_from_config。

* ../ceilometer/api/hooks.py: DBHook.```__init__``` --> get_connection

    ```
    ...
    class DBHook(hooks.PecanHook):
        def __init__(self):
            self.storage_connection = DBHook.get_connection('metering')
            self.event_storage_connection = DBHook.get_connection('event')
            self.alarm_stroage_connection = DBHook.get_connection('alarm')

            if (not self.storage_connection and not self.event_storage_connection and not self.alarm_storage_connection):
                raise Exception("Api failed to start. Failed to connect to database, purpose: %s" % ', '.join(['metering', 'event', 'alarm'])
        
        def before(self, state):
            state.request.storage_conn = self.storage_connection
            state.request.event_storage_conn = self.event_storage_connection
            state.request.alarm_storage_conn = self.alarm_storage_connection

        @staticmethod
        def get_connection(purpose):
            try:
                return storage.get_connection_from_config(cfg.CONF, purpose)
            except Exception as err:
                params = {"purpose": purpose, "err": err}
                LOG.exception(_LE("Failed to connect to db, purpose %(purpose)s retry later:%(err)s") % params)
    ...
    ```

下面再看storage.get_connection_from_config函数是如何执行的，函数storage.get_connection_from_config调用了函数storage.get_connection，而在函数storage.get_connection中重要的是: ```mgr = driver.DriverManger(namespace, engine_name)```，其中的driver为：```from stevedore import
driver```，namespace为：ceilometer.meterings.storage，engine_name为：mongodb。于是stevedore会到setup.cfg中查找相应的设置。

* ../ceilometer/storage/```__init__```.py: get_connection_from_config --> get_connection

    ```
    ...
    def get_connection_from_config(conf, purpose='metering'):
        retries = conf.database.max_retries

        @retrying.retry(wait_fixed=conf.database.retry_interval*1000,stop_max_attempt_number=retries if retries >= 0 else None)
        def __inner():
            if conf.database_connection:
                conf.set_override('connection', conf.database_connection,group='databse')
            namespace = 'ceilometer.%s.storage' % purpose
            url = (getattr(conf.database, '%s_connection' % purpose) or conf.database.connection)
            return get_connection(url, namespace)
        return __inner()

    def get_connection(url, namespace):
        connection_scheme = urlparse.urlparse(url).scheme
        engine_name = connection_scheme.split('+')[0]
        if engine_name == 'db2':
            import warnings
            warnings.simplefilter("always")
            import debtcollector
            debtcollector.deprecate("The DB2nosql driver is no longer supperted", version="Liberty", removal_version="N*-cycle")

        LOG.debug('looking for %(name)r driver in %(namespace)r', {'name': engine_name, 'namespace':namespace})
        mgr = driver.DriverManager(namespace, engine_name)
        return mgr.driver(url)
    ```

在../setup.cfg中能看到storage_conn会被赋值为ceilometer.storage.impl_monogodb:Connection，然后就会调用该Connection的```__init__```函数进行初始化。

* ../setup.cfg: mongodb = ceilometer.storage.impl_mongodb:Connection

    ```
    ceilometer.metering.storage = 
        log = ceilometer.storage.impl_log:Connection
        mongodb = ceilometer.storage.impl_mongodb:Connection
        mysql = ceilometer.storage.impl_sqlalchemy:Connection
        postgresql = ceilometer.storage.impl_sqlalchemy:Connection
        sqlite = ceilometer.storage.impl_sqlalchemy:Connection
        hbase = ceilometer.storage.impl_hbase:Connection
        db2 = ceilometer.storage.impl_db2:Connection
    ```

在ceilometer.storage.imple_mongodb.Connection类的初始化函数中会初始化数据库的连接，没有相应collection的情况下新建相应collection，设置ttl等。

* ../ceilometer/storage/impl_mongodb.py: Connection.```__init__```

    ```
    class Connection(pymongo_base.Connection):
        ...
        def __init__(self, url):
            self.conn = self.CONNECTION_POOL.connect(url)
            self.version = self.conn.server_info()['versionArray']
            if self.version < pymongo_utils.MINMUM_COMPATIBLE_MONGODB_VERSION:
                raise storage.StorageBadVersion("Need at least MongoDB %s" % pymongo_utils.MINIMUM_COMPATIBLE_MONGODB_VERSION)

            connection_options = pymongo.uri_parser.parse_uri(url)
            self.db = getattr(self.conn, connection_options['database'])
            if connection_options.get('username'):
                self.db.authenticate(connection_options['username'],connection_options['password'])

            self.upgrade()

        @staticmethod
        def update_ttl(ttl, ttl_index_name, index_field, coll):
            indexes = coll.index_information()
            if ttl <= 0:
                if ttl_index_name in indexes:
                    coll.drop_index(ttl_index_name)
                return 

            if ttl_index_name in indexes:
                return coll.database.command('collMod', coll.name, index={'keyPattern': {index_field: pymongo.ASCENDING),'expireAfterSeconds': ttl})

            coll.create_index([(index_field, pymongo.ASCENDING)],expireAfterSeconds=ttl,name=ttl_index_name)

        def upgrade(self):
            if 'resource' not in self.db.conn.collection_names():
                self.db.conn.create_collection('resource')
            if 'meter' not in self.db.conn.collection_names():
                self.db.conn.create_collection('meter')

            name_qualifier = dict(user_id='', project_id='project_')
            backgroud = dict(user_id=False, project_id=True)
            for primary in ['user_id', 'project_id']:
                name = 'meter_%sidx' % name_qualifier[primary]
                self.db.meter.create_index([
                    ('resource_id', pymongo.ASCENDING),
                    (primary, pymongo.ASCENDING),
                    ('counter_name', pymongo.ASCENDING),
                    ('timestamp', pymongo.ASCENDING),
                ], name=name, background=background[primary])

            self.db.meter.create_index([('timestamp', pymongo.DESCENDING)], name='timestamp_idx')

            self.db.resource.create_index([('user_id',pymongo.DESCENDING),('project_id', pymongo.DESCENDING),('last_sample_timestamp',pymongo.DESCENDING)],name='resource_user_project_timestamp',)
            self.db.resource.create_index([('last_sample_timestamp',pymongo.DESCENDING)],name='last_sample_timestamp_idx')

            ttl = cfg.CONF.database.metering_time_to_live
            self.update_ttl(ttl, 'meter_ttl', 'timestamp', self.db.meter)
            self.update_ttl(ttl, 'resource_ttl', 'last_sample_timestamp',self.db.resource)
        ...
    ```

#### get_meters的执行过程

这里有一个类的继承关系：object <-- storage.base.Connection <-- storage.pymongo_base.Connection <-- storage.impl_mongodb.Connection，关于storage.impl_mongodb.Connection需要讲的都在上面提到了。

在storage.base.Connection中给出了一些函数定义，但都没有具体的实现。

* ../ceilometer/storage/base.py

    ```
    class Connection(object):
        ...
        def __init__(self, url):
            pass

        @staticmethod
        def upgrade():
        ...
    ```

在storage.pymongo_base.Connection中给出了get_meters的定义，在这里我们就可以真正的看到查询数据库的语句了: self.db.resource.find(q)，下面我们还要解释一下语句models.Meter。

* ../ceilometer/storage/pymongo_base.py: Connection.get_meters

    ```
    ...
    class Connection(base.Connection):
        ...
        def get_meters(self, user=None, project=None, resource=None, source=None, metaquery=None, limit=None):
            if limit == 0:
                return

                metaquery = pymongo_utils.improve_keys(metaquery, metaquery=True) or {}

            q = {}
            if user is not None
                q['user_id'] = user
            if project is not None:
                q['project_id'] = project
            if resource is not None:
                q['_id'] = resource
            if source is not None:
                q['source'] = source
            q.update(metaquery)

            count = 0
            for r in self.db.resource.find(q):
                for r_meter in r['meter']:
                    if limit and count >= limt:
                        return
                    else:
                        count += 1
                    yield models.Meter(
                        name=r_meter['counter_name'],
                        type=r_meter['counter_type']
                        unit=r_meter.get('counter_unit', ''),
                        resource_id=r['_id'],
                        porject_id=r['project_id'],
                        source=r['source'],
                        user_id=r['user_id'],
                    )
        ...
    ```

这里又有一个类的继承关系：object <-- storage.base.Model <-- storage.models.Meter。
这里其实就是把查询到的值和键做个对应的设置。

* ../ceilometer/storage/base.py: Model

    ```
    class Model(object):
        def __init__(self, **kwds):
            self.fields = list(kwds)
            for k, v in six.iteritems(kwds):
                setattr(self, k, v)
        def as_dict(self):
            d = {}
            for f in self.fields:
                v = getattr(self, f)
                if isinstance(v, Model):
                    v = v.as_dice()
                elif isinstance(v, list) and v and isinstance(v[0], Model):
                    v = [sub.as_dict() for sub in v]
                d[f] = v
            return d
        ...
    ```

* ../ceilometer/storage/models.py

    ```
    ...
    class Meter(base.Model):
        def __init__(self, name, type, unit, resource_id, project_id, source, user_id):
            base.Model.__init__(self,
                                name=name,
                                type=type,
                                unit=unit,
                                resource_id=resource_id,
                                project_id=project_id,
                                source=source,
                                user_id=user_id,
                                )
    ...
    ```

### Meter.from_db_model(m)的执行过程

pecan.request.storage_conn.get_meters的执行过程讲解完毕，终于要到最后一部分了:Meter.from_db_model(m)。

这里有个类的继承关系：wsme.types.Base <-- wsme.types.DynamicBase <-- api.controllers.v2.base.Base <-- api.v2.meters.Meter

后面api.v2.meters.Meter的初始化函数调用了```wsme.types.Base.__init__```函数。
对于wsme和pecan有的时候看源码比看文档来得快而准确。

* wsme.types

    ```
    ...
    class Base(six.with_metaclass(BaseMeta)):
        def __init__(self, **kw):
            for key, value in kw.items()
                if hasattr(self, key):
                    setattr(self, key, value)
    ...
    class DynamicBase(Base):
        @classmethod
        def add_attributes(cls, **attrs):
            for n,t in attrs.items():
                setattr(cls, n, t)
            cls.__registry__.reregister(cls)
    ...
    ```

函数Meter.from_db_model定义在ceilometer.api.controllers.v2.base:Base中，函数Meter.from_db_model是把返回的ceilometer.storage.models.Meter实例，
进一步做一些简单的处理。

* ../ceilometer/api/controllers/v2/base.py

    ```
    ...
    class Base(wtypes.DynamicBase):
        @classmethod
        def from_db_model(cls, m):
            return cls(**(m.as_dict()))

        @classmethod
        def from_db_and_links(cls, m, links):
            return cls(links=links, **(m.as_dict()))

        def as_dict(self, db_model):
            valid_keys = inspect.getargspec(db_model.__init__)[0]
            if 'self' in valid_keys:
                valid_keys.remove('self')
            return self.as_dict_from_keys(valid_keys)

        def as_dict_from_keys(self, keys):
            return dict((k, getattr(self, keys))for k in keys if hasattr(self, k) and getattr(self, k) != wsme.Unset)
    ...
    ```

Meter是在ceilometer.api.v2.meters中定义的，Meter中定义它的的类属性:name, type, unit, resource_id, project_id, user_id, source, meter_id；Meter中还定义了初始化函数```__init__```，该初始化函数主要是构造meter_id和调用wsme.types.Base的初始化函数```__init__```。

* ../ceilometer/api/v2/meters.py

    ```
    ...
    class Meter(base.Base):
        name = wtypes.text
        "The unique name for the meter"

        type = wtypes.Enum(str, *sample.TYPES)
        "The meter type (see :ref:`measurements`)"

        unit = wtypes.text
        "The unit of measure"

        resource_id = wtypes.text
        "The ID of the :class:`Resource` for which the measurements are taken"

        project_id = wtypes.text
        "The ID of the project or tenant that owns the resource"

        user_id = wtypes.text
        "The ID of the user who last triggered an update to the resource"

        source = wtypes.text
        "The ID of the source that identifies where the meter comes from"

        meter_id = wtypes.text
        "The unique identifier for the meter"

        def __init__(self, **kwargs):
            meter_id = '%s+%s' % (kwargs['resource_id'], kwargs['name'])
            meter_id = base64.b64encode(meter_id.encode('urg-8'))
            kwargs['meter_id'] = meter_id
            super(Meter, self).__init__(**kwargs)
        ...
    ...
    ```

## 最后

还是有很多细节问题没有弄清，需要继续研究
