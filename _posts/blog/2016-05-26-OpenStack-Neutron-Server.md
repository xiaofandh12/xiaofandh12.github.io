---
layout: post
title: OpenStack Neutron -- neutron-server服务的启动流程
description: OpenStack, Neutron, neutron-server
category: blog.bak
---

### neutron-server服务启动的大致流程

* setup.cfg -- neutron-server = neutron.cmd.eventlet.server:main
* neutron/cmd/eventlet/server/__init__.py -- def main()
* neutron/server/__init__.py -- def boot_server(server_func)
* neutron/cmd/eventlet/server/__init__.py -- def _main_neutron_server()
* neutron/server/wsgi_pecan.py -- def pecan_wsgi_server()
* neutron/pecan_wsgi/app.py -- def setup_app(*args, **kwargs)
* neutron/service.py -- def run_wsgi_app(app)
* neutron/server/wsgi_eventlet.py -- def start_api_and_rpc_workers(neutron_api)
* neutron/service.py -- def service_rpc()
* neutron/service.py -- def start_plugin_workers()
