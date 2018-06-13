---
layout:     post
title:      neutron web server启动过程
subtitle:
date:       2018-2-15
author:     dodoYuan
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - neutron
---

函数入口为：

cmd.server中的main函数
```
def main():
    server.boot_server(wsgi_eventlet.eventlet_wsgi_server)
```
调用的函数原型为：
```
def eventlet_wsgi_server():
    neutron_api = service.serve_wsgi(service.NeutronApiService)
    start_api_and_rpc_workers(neutron_api)
```
第一句启动了一个web server；第二句就是创建一个RPC server。先分析web server启动过程。
#### 1、web server启动过程
serve_wsgi函数传入参数NeutronApiService，NeutronApiService是一个类，该类继承自WsgiService。
```
class NeutronApiService(WsgiService):
    """Class for neutron-api service."""
    def __init__(self, app_name):
        profiler.setup('neutron-server', cfg.CONF.host)
        super(NeutronApiService, self).__init__(app_name)

    @classmethod
    def create(cls, app_name='neutron'):
        # Setup logging early
        config.setup_logging()
        service = cls(app_name)
        return service
```
WsgiService是所有neutron api server的父类，里面定义了start等重要方法。
```
class WsgiService(object):
    """Base class for WSGI based services.

    For each api you define, you must also define these flags:
    :<api>_listen: The address on which to listen
    :<api>_listen_port: The port on which to listen

    """

    def __init__(self, app_name):
        self.app_name = app_name
        self.wsgi_app = None

    def start(self):
        self.wsgi_app = _run_wsgi(self.app_name)

    def wait(self):
        self.wsgi_app.wait()
```
上面分析了service.serve_wsgi(service.NeutronApiService)中传入的函数基本实现，接下来看看serve_wsgi函数，serve_wsgi调用NeutronApiService的create类方法，并启动。
```
def serve_wsgi(cls):

    try:
        service = cls.create()
        service.start()
    except Exception:
        with excutils.save_and_reraise_exception():
            LOG.exception('Unrecoverable error: please check log '
                          'for details.')

    registry.publish(resources.PROCESS, events.BEFORE_SPAWN, service)
    return service
```
所以最终定为到service.start()函数，返回WsgiService类，发现关键代码为：
```
def _run_wsgi(app_name):
    app = config.load_paste_app(app_name)
    if not app:
        LOG.error('No known API applications configured.')
        return
    return run_wsgi_app(app)
```
百转千回，发现最终启动APP的是run_wsgi_app(app)函数，这里需要先加载APP（APP的名字使用默认neutron）。
load_paste_app函数在common.config.py文件中
```
def load_paste_app(app_name):
    """Builds and returns a WSGI app from a paste config file.

    :param app_name: Name of the application to load
    """
    loader = wsgi.Loader(cfg.CONF)
    app = loader.load_app(app_name)
    return app
```
neutron源码中不支持from oslo_service import wsgi ，在OSLO项目中，找到程序如下：
```
class Loader(object):
    """Used to load WSGI applications from paste configurations."""

    def __init__(self, conf):
        """Initialize the loader, and attempt to find the config.
        :param conf: Application config
        :returns: None
        """
        conf.register_opts(_options.wsgi_opts)
        self.config_path = None

        config_path = conf.api_paste_config
        if not os.path.isabs(config_path):
            self.config_path = conf.find_file(config_path)
        elif os.path.exists(config_path):
            self.config_path = config_path

        if not self.config_path:
            raise ConfigNotFound(path=config_path)

    def load_app(self, name):
        """Return the paste URLMap wrapped WSGI application.
        :param name: Name of the application to load.
        :returns: Paste URLMap object wrapping the requested application.
        :raises: PasteAppNotFound
        """
        try:
            LOG.debug("Loading app %(name)s from %(path)s",
                      {'name': name, 'path': self.config_path})
            return deploy.loadapp("config:%s" % self.config_path, name=name)
        except LookupError:
            LOG.exception("Couldn't lookup app: %s", name)
            raise PasteAppNotFound(name=name, path=self.config_path)
```
上面程序主要用来加载一个 WSGI app，最终落实的函数是paste.deploy.loadapp，不在详细介绍。
接下来是run_wsgi_app函数，启动加载好的APP
```
def run_wsgi_app(app):
    server = wsgi.Server("Neutron")
    server.start(app, cfg.CONF.bind_port, cfg.CONF.bind_host,
                 workers=_get_api_workers())
    LOG.info("Neutron service started, listening on %(host)s:%(port)s",
             {'host': cfg.CONF.bind_host, 'port': cfg.CONF.bind_port})
    return server
```
思路很清楚，先加载一个Server实例，然后传入该APP并启动。其中Server函数为
```
class Server(object):
    """Server class to manage multiple WSGI sockets and applications."""

    def __init__(self, name, num_threads=None, disable_ssl=False):
        # Raise the default from 8192 to accommodate large tokens
        eventlet.wsgi.MAX_HEADER_LINE = CONF.max_header_line
        self.num_threads = num_threads or CONF.wsgi_default_pool_size
        self.disable_ssl = disable_ssl
        # Pool for a greenthread in which wsgi server will be running
        self.pool = eventlet.GreenPool(1)
        self.name = name
        self._server = None
        # A value of 0 is converted to None because None is what causes the
        # wsgi server to wait forever.
        self.client_socket_timeout = CONF.client_socket_timeout or None
        if CONF.use_ssl and not self.disable_ssl:
            sslutils.is_enabled(CONF)

    def start(self, application, port, host='0.0.0.0', workers=0):
        """Run a WSGI server with the given application."""
        self._host = host
        self._port = port
        backlog = CONF.backlog

        self._socket = self._get_socket(self._host,
                                        self._port,
                                        backlog=backlog)

        self._launch(application, workers)
```
上面函数用来启动一个web server，在port和host上监听，
```
    def _launch(self, application, workers=0):
        service = WorkerService(self, application, self.disable_ssl, workers)
        if workers < 1:
            # The API service should run in the current process.
            self._server = service
            # Dump the initial option values
            cfg.CONF.log_opt_values(LOG, logging.DEBUG)
            service.start()
            systemd.notify_once()
        else:
            # dispose the whole pool before os.fork, otherwise there will
            # be shared DB connections in child processes which may cause
            # DB errors.
            api.context_manager.dispose_pool()
            # The API service runs in a number of child processes.
            # Minimize the cost of checking for child exit by extending the
            # wait interval past the default of 0.01s.
            self._server = common_service.ProcessLauncher(
                cfg.CONF, wait_interval=1.0, restart_method='mutate')
            self._server.launch_service(service,
                                        workers=service.worker_process_count)
```
_launch函数又初始化了WorkerService类，传入了server自身和application实例。并且根据workers来决定是在当前进程还是另外启动新进程来运行web server
```
class WorkerService(neutron_worker.BaseWorker):
    """Wraps a worker to be handled by ProcessLauncher"""
    def __init__(self, service, application, disable_ssl=False,
                 worker_process_count=0):
        super(WorkerService, self).__init__(worker_process_count)

        self._service = service
        self._application = application
        self._disable_ssl = disable_ssl
        self._server = None

    def start(self):
        super(WorkerService, self).start()
        # When api worker is stopped it kills the eventlet wsgi server which
        # internally closes the wsgi server socket object. This server socket
        # object becomes not usable which leads to "Bad file descriptor"
        # errors on service restart.
        # Duplicate a socket object to keep a file descriptor usable.
        dup_sock = self._service._socket.dup()
        if CONF.use_ssl and not self._disable_ssl:
            dup_sock = sslutils.wrap(CONF, dup_sock)
        self._server = self._service.pool.spawn(self._service._run,
                                                self._application,
                                                dup_sock)
```
代码有点乱，梳理一下，在 run_wsgi_app函数中，实例化了wsgi.Server("Neutron")，调用该函数的start，传入socket相关的参数和workers数，该start函数完成的功能有两个，建立一个socket服务端监听，并调用_launch(application, workers)，而_launch函数又实例化service = WorkerService(self, application, self.disable_ssl, workers)，并根据workers数目确定在当前进程启动还是在新增加进程启动，产生了一个分支。所以说到现在还没有启动web server。
在当前进程启动的话直接调用WorkerService的start函数，通过pool.spawn生成了一个新的协程，用来运行如下函数：
```
    def _run(self, application, socket):
        """Start a WSGI server in a new green thread."""
        eventlet.wsgi.server(socket, application,
                             max_size=self.num_threads,
                             log=LOG,
                             keepalive=CONF.wsgi_keep_alive,
                             log_format=CONF.wsgi_log_format,
                             socket_timeout=self.client_socket_timeout)
```
在新进程中启动web server方式不做多介绍。

