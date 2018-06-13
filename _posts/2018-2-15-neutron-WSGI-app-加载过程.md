---
layout:     post
title:      neutron WSGI application 加载过程
subtitle:
date:       2018-2-17
author:     dodoYuan
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - openstack neutron
---

#### 1、WSGI接口介绍
进入正题之前，先复习下WSGI
>WSGI的出发点：底层代码由专门的服务器软件实现，我们用Python专注于生成HTML文档。因为我们不希望接触到TCP连接、HTTP原始请求和响应格式，所以，需要一个统一的接口，让我们专心用Python编写Web业务。

WSGI接口定义非常简单，它只要求Web开发者实现一个函数，就可以响应HTTP请求。我们来看一个最简单的Web版本的“Hello, web!”：
```
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return '<h1>Hello, web!</h1>'
```
上面的application()函数就是符合WSGI标准的一个HTTP处理函数，它接收两个参数：
environ：一个包含所有HTTP请求信息的dict对象；
start_response：一个发送HTTP响应的函数。
在application()函数中，调用：
```
start_response('200 OK', [('Content-Type', 'text/html')])
```
就发送了HTTP响应的Header，注意Header只能发送一次，也就是只能调用一次start_response()函数。start_response()函数接收两个参数，一个是HTTP响应码，一个是一组list表示的HTTP Header，每个Header用一个包含两个str的tuple表示。
通常情况下，都应该把Content-Type头发送给浏览器。其他很多常用的HTTP Header也应该发送。
然后，函数的返回值'<h1>Hello, web!</h1>'将作为HTTP响应的Body发送给浏览器。

简单总结下，WSGI将Web组件分为三类：

* 服务器(WSGI Server)，负责接收HTTP请求，并封装一系列的环境变量。
* 中间件(WSGI Middleware)，负责将HTTP请求路由给不同的应用对象，并将处理结果返回给WSGI Server。从服务端看，中间件就是个WSGI应用，而从应用端看，中间件相当于服务端。
* 应用程序(WSGI Application)，是可被调用的Python对象，一般接收两个参数，environ和start_response（回调函数）。

当一个请求发送（转发）到WSGI Server时，Server会为应用端准备上下文信息和回调函数，应用端处理完后，便使用提供的回调函数返回相应的请求结果。其中中间件则作为服务端和应用端之间交互的一种包装，经过不同的中间件，便具有不同的功能，如URL分发、权限认证等，不同中间件的组合形成和WSGI的框架。虽然感觉很复杂，但Openstack中使用Paset的Deploy组件来完成WSGI服务器和应用的构建，这一切就变得简单多了。

一个简单的WSGI示意：
```
# server.py
# 从wsgiref模块导入:
from wsgiref.simple_server import make_server
# 导入我们自己编写的application函数:
from hello import application

# 创建一个服务器，IP地址为空，端口是8000，处理函数是application:
httpd = make_server('', 8000, application)
print "Serving HTTP on port 8000..."
# 开始监听HTTP请求:
httpd.serve_forever()
```
然后就可以在服务器端打开相应的界面。
[以上参考廖雪峰老师WSGI接口](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386832689740b04430a98f614b6da89da2157ea3efe2000)

#### 2、paste deploy相关
paste deploy是用来发现和配置WSGI应用的一套系统，对于WSGI应用的使用者而言，可以方便地从配置文件汇总加载WSGI应用（loadapp）；对于WSGI应用的开发人员而言，只需要给自己的应用提供一套简单的入口点即可。
PasteDeploy配置文件很关键，由若干section组成，section的声明格式如下：
` [type:name] `
其中，方括号括起的section声明一个新section的开始，section的声明由两部分组成，section的类型（type）和section的名称（name），如：[app:main]等。section的type可以有：app、composite、filter、pipeline、filter-app等。
每个section中具体配置项的格式就是基本的ini格式： key = value ，所有从PasteDeploy配置文件中提取的参数键、值都以字符串的形式传入底层实现。
简单示例：
```
   [composite:main]
　　use = egg:Paste#urlmap
　　/ = home
　　/blog = blog
　　/wiki = wiki
　　/cms = config:cms.ini

　　[app:home]
　　use = egg:Paste#static
　　document_root = %(here)s/htdocs

　　[app:wiki] 
　　use = call:mywiki.main:application 
　　database = sqlite:/home/me/wiki.db

　　[filter-app:blog] 
　　use = egg:Authentication#auth 
　　next = blogapp 
　　roles = admin 
　　htpasswd = /home/me/users.htpasswd 

　　[app:blogapp]
　　use = egg:BlogApp
　　database = sqlite:/home/me/blog.db

　　[app:main]
　　use = egg:MyEgg
　　filter-with = printdebug

　　[filter:printdebug] 
　　use = egg:Paste#printdebug 

　　[pipeline:main]
　　pipeline = filter1 filter2 filter3 app
　　...
```
* Type = composite（组合应用），组合应用由若干WSGI应用组成，composite为这些应用提供更高一层的分配工作。Composite类型的section将URL请求分配给其他的WSGI应用。 use = egg:Paste#urlmap 意味着使用 Paste 包中的 urlmap 应用。urlmap是Paste提供的一套通用的composite应用，作用就是根据用户请求的URL前缀，将用户请求映射到对应的WSGI应用上去。这里的WSGI应用有："home", "blog", "wiki" 和 "config:cms.ini"。
* app类型的section声明一个具体的WSGI应用。调用哪个python module中的app代码则由的use后的值指定。这里的 egg:Paste#static 是另一个简单应用，作用仅仅是呈现静态页面。它接收了一个配置项： document_root ，后面的值可以从全局配置DEFAULT(大小写敏感)中提取，提取方法s是使用变量替换：比如%(var_name)s 的形式。这里 %(here)s 的意思是这个示例配置文件所在的目录。

参考
https://tommylike.github.io/openstack/2016/08/03/Paste-Deploy-Introduce/
#### 3、neutron api-paste.ini配置文件加载app分析
这里稍微深入理解下WSGI应用加载过程，也就是load_paste_app(app_name)函数，loader = wsgi.Loader(cfg.CONF)和app = loader.load_app(app_name)
相关源码超出了neutron的范围，在OSLO的server模块中，关键代码如下：
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
deploy.loadapp("config:%s" % self.config_path, name=name)是关键，就是从一个paste config file中构建一个WSGI application。config_path = conf.api_paste_config，可以定位到etc/api-paste.ini文件。
```
[composite:neutron]
use = egg:Paste#urlmap
/: neutronversions_composite
/v2.0: neutronapi_v2_0

[composite:neutronapi_v2_0]
use = call:neutron.auth:pipeline_factory
noauth = cors http_proxy_to_wsgi request_id catch_errors extensions neutronapiapp_v2_0
keystone = cors http_proxy_to_wsgi request_id catch_errors authtoken keystonecontext extensions neutronapiapp_v2_0
...
```
不去深究deploy.loadapp实现，但我们知道api-paste.ini文件对应的是一个个WSGI application，通过事先约定好的规则，对HTTP的一系列URL请求进行分发到对应的app上。
[composite:neutron]通过urlmap应用提取URL特征，以'/'开头的交由neutronversions_composite处理，以‘/v2.0’开头的url交由neutronapi_v2_0处理。这两个section都涉及到pipeline，我们对[composite:neutronapi_v2_0]做详细分析。
[composite:neutronapi_v2_0]使用neutron.auth:pipeline_factory函数处理http请求，以key/value字典形式传入noauth和keystone两个参数。
```
def pipeline_factory(loader, global_conf, **local_conf):
    """Create a paste pipeline based on the 'auth_strategy' config option."""
    pipeline = local_conf[cfg.CONF.auth_strategy]
    pipeline = pipeline.split()
    filters = [loader.get_filter(n) for n in pipeline[:-1]]
    app = loader.get_app(pipeline[-1])
    filters.reverse()
    for filter in filters:
        app = filter(app)
    return app
```
cfg.CONF.auth_strategy中设定的是keystone，因此pipeline = ‘cors http_proxy_to_wsgi request_id catch_errors authtoken keystonecontext extensions neutronapiapp_v2_0’，filters = [loader.get_filter(n) for n in pipeline[:-1]]这句话通过调用loader，获取一系列的filter，注意到pipeline 中其实是一些列section name，即
```
[filter:request_id]
paste.filter_factory = oslo_middleware:RequestId.factory

[filter:catch_errors]
paste.filter_factory = oslo_middleware:CatchErrors.factory

[filter:cors]
paste.filter_factory = oslo_middleware.cors:filter_factory
oslo_config_project = neutron

[filter:http_proxy_to_wsgi]
paste.filter_factory = oslo_middleware.http_proxy_to_wsgi:HTTPProxyToWSGI.factory
```
所以loader.get_filter会将pipeline中存储的所有filter type的section所对应的具备filter特征的WSGI application加载。
app = loader.get_app(pipeline[-1])加载最后一个app，也就是 app = neutronapiapp_v2_0，接下来将所有的filter倒序排列，后面三句就是将app一层一层加上过滤条件，首先加上的是extensions ，然后是keystonecontext，最后是cors ，这也是上面需要翻转的原因，因为首先要加上最内层的filter。
#### 4、WSGI整体流程分析
梳理整个流程，在前面web server启动流程我们分析过最终通过协程运行的是如下函数
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
短短一行代码就构建了一个包含WSGI application的web server，当一个http到达这个web server，先经过urlmap处理，如果是以‘/2.0’开头，那么在交给cors、http_proxy_to_wsgi，一步一步传递到 extensions ，最后再交给neutronapiapp_v2_0处理。
所以说ini配置文件很强大，实现了对URL的分发以及对处理流程pipeline化。
但上述还只是机制上的一些处理，没有说明具体应用是如何运行的。坐稳咯，开始出发了~
我们先看最关键的，也就是neutronapiapp_v2_0 app的实现。
```
[app:neutronapiapp_v2_0]
paste.app_factory = neutron.api.v2.router:APIRouter.factory
```
如上，函数定位到:
```
def APIRouter(**local_config):
    return pecan_app.v2_factory(None, **local_config)

def _factory(global_config, **local_config):
    return pecan_app.v2_factory(global_config, **local_config)

setattr(APIRouter, 'factory', _factory)
```
上来就很晕，定位函数为
```
def v2_factory(global_config, **local_config):
    # Processing Order:
    #   As request enters lower priority called before higher.
    #   Response from controller is passed from higher priority to lower.
    app_hooks = [
        hooks.UserFilterHook(),  # priority 90
        hooks.ContextHook(),  # priority 95
        hooks.ExceptionTranslationHook(),  # priority 100
        hooks.BodyValidationHook(),  # priority 120
        hooks.OwnershipValidationHook(),  # priority 125
        hooks.QuotaEnforcementHook(),  # priority 130
        hooks.NotifierHook(),  # priority 135
        hooks.QueryParametersHook(),  # priority 139
        hooks.PolicyHook(),  # priority 140
    ]
    app = pecan.make_app(root.V2Controller(),
                         debug=False,
                         force_canonical=False,
                         hooks=app_hooks,
                         guess_content_type_from_ext=True)
    startup.initialize_all()
    return app
```
所以pecan.make_app还得了解下，这个函数返回了最终的APP，因此可以猜测pecan.make_app肯定实现了请求和相应的处理逻辑。
http://huntxu.github.io/2015-12-03-neutron-in-half-an-hour.html