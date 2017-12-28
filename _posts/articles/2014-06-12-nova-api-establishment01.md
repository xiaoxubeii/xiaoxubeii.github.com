---
layout: post
title: "nova-api建立流程01"
description: ""
category: articles
tags: [nova, nova-api]
---
最近在面试的时候，被问到/etc/nova/api-paste.ini的用途。当时也是一知半解，只知道可能是nova-api启动时的配置文件。回来后仔细看了一下，才发现不是这么简单。

#Paste Deployment简介

Paste Deployment是一套发现和部署WSGI app和server的系统，它实际上屏蔽了app的具体实现细节，这使得用户可以简单快速的部署WSGI app和server。
通过pip命令就可以直接安装：

    # pip install PasteDeploy
    
下面是一个典型的paste deployment配置文件：

    #composite section会将请求dispatch到其他的app中，这里是根据url
    [composite:main]
    use = egg:Paste#urlmap
    /firstapp = firstpipe
    /secondapp = secondapp
    
    #这是一个app factory，它会生成一个callable的app
    [app:firstapp]
    use = app:FirstApp.factory
    
    [app:secondapp]
    use = app:SecondApp.factory
    
    #这是一个filter factory，它会生成一个filter
    [filter:firstfilter]
    use = filter:FirstFilter.factory
    
    [filter:secondfilter]
    use = filter:SecondFilter.factory
    
    #pipeline用来将多个filter和app绑定到一起，最后返回过滤后的app
    [pipeline:firstpipe]
    pipline = firstapp secondfilter firstapp

相关的代码如下：

    from webob import Response
    from wsgi import Router
    from wsgi import Resource
    
    class FirstApp():
        def __call__(self, environ, start_response):
            res = Response()
            res.status = '200 OK'
            res.content_type = 'text/plain'
            content = []
            content.append('I am first app')
            res.body = '\n'.join(content)
            return res(environ, start_response)
        
        @classmethod
        def factory(cls, global_conf, **kwargs):
            return cls()
    
    class FirstFilter():
        def __init__(self, app):
            self.app = app
            
        def __call__(self, environ, start_response):
            print 'first filter'
            return self.app(environ, start_response)
    
        @classmethod
        def factory(cls, global_conf, **kwargs):
            return cls 
        
    class SecondFilter():
        def __init__(self, app):
            self.app = app
            
        def __call__(self, environ, start_response):
            print 'second filter'
            return self.app(environ, start_response)
    
        @classmethod
        def factory(cls, global_conf, **kwargs):
            return cls
            
    class SecondApp():
        def __call__(self, environ, start_response):
            res = Response()
            res.status = '200 OK'
            res.content_type = 'text/plain'
            content = []
            content.append('I am second app')
            res.body = '\n'.join(content)
            return res(environ, start_response)
        
        @classmethod
        def factory(cls, global_conf, **kwargs):
            return cls()

这样的话，可以通过下面的方式启动一个server和app：

    from paste.deploy import loadapp
    from wsgiref.simple_server import make_server
    
    appname = "main"
    paste_path = "/etc/wsgiapp/paste.ini"
    wsgi_app = loadapp("config:%s" % paste_path, appname)
    server = make_server('localhost', 8080, wsgi_app)
    server.serve_forever()

具体关于Paste Deployment中composite、app、filter和pipeline这些section的介绍可以见[Paste Deploy v1.5.0 documentatin][1]。

#Routes
在OpenStack的REST API service中还会遇到routes的概念。python中的routes是仿照Ruby on Rails的routing，主要是实现路由映射的功能。
通过pip命令就可以直接安装：

    # pip install Routes

它使用起来也是非常方便：

    map = mapper()
    map.connect("home", "/", controller="main", action="index")

这样如果请求的url是http://localhost:8080/，routes就会路由到controller为main的index函数中。

##RESTful services
利用routes中的map.resources还可以方便的构建一个RESTful web services。

    map.resource("message", "messages")

    # 上面的代码相当于手动建立了下面这些路由规则
    map.connect("messages", "/messages",
        controller="messages", action="create",
        conditions=dict(method=["POST"]))
    map.connect("messages", "/messages",
        controller="messages", action="index",
        conditions=dict(method=["GET"]))
    map.connect("formatted_messages", "/messages.{format}",
        controller="messages", action="index",
        conditions=dict(method=["GET"]))
    map.connect("new_message", "/messages/new",
        controller="messages", action="new",
        conditions=dict(method=["GET"]))
    map.connect("formatted_new_message", "/messages/new.{format}",
        controller="messages", action="new",
        conditions=dict(method=["GET"]))
    map.connect("/messages/{id}",
        controller="messages", action="update",
        conditions=dict(method=["PUT"]))
    map.connect("/messages/{id}",
        controller="messages", action="delete",
        conditions=dict(method=["DELETE"]))
    map.connect("edit_message", "/messages/{id}/edit",
        controller="messages", action="edit",
        conditions=dict(method=["GET"]))
    map.connect("formatted_edit_message", "/messages/{id}.{format}/edit",
        controller="messages", action="edit",
        conditions=dict(method=["GET"]))
    map.connect("message", "/messages/{id}",
        controller="messages", action="show",
        conditions=dict(method=["GET"]))
    map.connect("formatted_message", "/messages/{id}.{format}",
        controller="messages", action="show",
        conditions=dict(method=["GET"]))
        
这样会建立如下的转换规则：

    GET    /messages        => messages.index()    => url("messages")
    POST   /messages        => messages.create()   => url("messages")
    GET    /messages/new    => messages.new()      => url("new_message")
    PUT    /messages/1      => messages.update(id) => url("message", id=1)
    DELETE /messages/1      => messages.delete(id) => url("message", id=1)
    GET    /messages/1      => messages.show(id)   => url("message", id=1)
    GET    /messages/1/edit => messages.edit(id)   => url("edit_message", id=1)

#Nova-api WSGI service

##启动
OpenStack中的service有两种：WSGIService和Service，所有的service都在nova/cmd中。nova-api的启动代码在nova/cmd/api.py中：

    def main():
        config.parse_args(sys.argv)
        logging.setup("nova")
        utils.monkey_patch()
    
        gmr.TextGuruMeditation.setup_autorun(version)
    
        launcher = service.process_launcher()
        for api in CONF.enabled_apis:
            should_use_ssl = api in CONF.enabled_ssl_apis
            if api == 'ec2':
                server = service.WSGIService(api, use_ssl=should_use_ssl,
                                             max_url_len=16384)
            else:
                server = service.WSGIService(api, use_ssl=should_use_ssl)
            launcher.launch_service(server, workers=server.workers or 1)
        launcher.wait()

service.WSGIService()负责启动WSGI server。进入service.WSGIService()，在nova/service.py里，\_\_init\_\_()的代码如下：

    class WSGIService(object):
        """Provides ability to launch API from a 'paste' configuration."""
    
        def __init__(self, name, loader=None, use_ssl=False, max_url_len=None):
            """Initialize, but do not start the WSGI server.
    
            :param name: The name of the WSGI server given to the loader.
            :param loader: Loads the WSGI application using the given name.
            :returns: None
    
            """
            self.name = name
            self.manager = self._get_manager()
            self.loader = loader or wsgi.Loader()
            self.app = self.loader.load_app(name)
            self.host = getattr(CONF, '%s_listen' % name, "0.0.0.0")
            self.port = getattr(CONF, '%s_listen_port' % name, 0)
            self.workers = (getattr(CONF, '%s_workers' % name, None) or
                            utils.cpu_count())
            if self.workers and self.workers < 1:
                worker_name = '%s_workers' % name
                msg = (_("%(worker_name)s value of %(workers)s is invalid, "
                         "must be greater than 0") %
                       {'worker_name': worker_name,
                        'workers': str(self.workers)})
                raise exception.InvalidInput(msg)
            self.use_ssl = use_ssl
            self.server = wsgi.Server(name,
                                      self.app,
                                      host=self.host,
                                      port=self.port,
                                      use_ssl=self.use_ssl,
                                      max_url_len=max_url_len)
            # Pull back actual port used
            self.port = self.server.port
            self.backdoor_port = None
        ...
        
注意以下几句：

    self.loader = loader or wsgi.Loader()
    self.app = self.loader.load_app(name)

在初始化WSGIService时，会传入一个loader，并利用这个loader去创建一个app。如果没有传入，就使用wsgi.Loader。

    self.server = wsgi.Server(name,
                              self.app,
                              host=self.host,
                              port=self.port,
                              use_ssl=self.use_ssl,
                              max_url_len=max_url_len)
                        
在这一步，会利用这个app及其他参数来创建一个server。最后，是在nova/cmd/api.py的main()的launcher.launch_service()中去启动这个service（具体的启动过程我会有专门的一篇文章介绍）：

    def main():
        ...
        for api in CONF.enabled_apis:
            ...
            launcher.launch_service(server, workers=server.workers or 1)
        ...

##api-paste.ini
在nova-api启动时，会load一个app，这个app是依据api-paste.ini创建的，其实它就是一个普通的Paste Deployment。nova中的api-paste.ini的示例可以见etc/nova/api-paste.ini：

    ############
    # Metadata #
    ############
    [composite:metadata]
    use = egg:Paste#urlmap
    /: meta
    
    [pipeline:meta]
    pipeline = ec2faultwrap logrequest metaapp
    
    [app:metaapp]
    paste.app_factory = nova.api.metadata.handler:MetadataRequestHandler.factory
    
    #######
    # EC2 #
    #######
    
    [composite:ec2]
    use = egg:Paste#urlmap
    /services/Cloud: ec2cloud
    
    [composite:ec2cloud]
    use = call:nova.api.auth:pipeline_factory
    noauth = ec2faultwrap logrequest ec2noauth cloudrequest validator ec2executor
    keystone = ec2faultwrap logrequest ec2keystoneauth cloudrequest validator ec2executor
    
    [filter:ec2faultwrap]
    paste.filter_factory = nova.api.ec2:FaultWrapper.factory
    
    [filter:logrequest]
    paste.filter_factory = nova.api.ec2:RequestLogging.factory
    
    [filter:ec2lockout]
    paste.filter_factory = nova.api.ec2:Lockout.factory
    
    [filter:ec2keystoneauth]
    paste.filter_factory = nova.api.ec2:EC2KeystoneAuth.factory
    
    [filter:ec2noauth]
    paste.filter_factory = nova.api.ec2:NoAuth.factory
    
    [filter:cloudrequest]
    controller = nova.api.ec2.cloud.CloudController
    paste.filter_factory = nova.api.ec2:Requestify.factory
    
    [filter:authorizer]
    paste.filter_factory = nova.api.ec2:Authorizer.factory
    
    [filter:validator]
    paste.filter_factory = nova.api.ec2:Validator.factory
    
    [app:ec2executor]
    paste.app_factory = nova.api.ec2:Executor.factory
    
    #############
    # OpenStack #
    #############
    
    [composite:osapi_compute]
    use = call:nova.api.openstack.urlmap:urlmap_factory
    /: oscomputeversions
    /v1.1: openstack_compute_api_v2
    /v2: openstack_compute_api_v2
    /v3: openstack_compute_api_v3
    
    [composite:openstack_compute_api_v2]
    use = call:nova.api.auth:pipeline_factory
    noauth = faultwrap sizelimit noauth ratelimit osapi_compute_app_v2
    keystone = faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
    keystone_nolimit = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v2
    
    [composite:openstack_compute_api_v3]
    use = call:nova.api.auth:pipeline_factory_v3
    noauth = faultwrap sizelimit noauth_v3 osapi_compute_app_v3
    keystone = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v3
    
    [filter:faultwrap]
    paste.filter_factory = nova.api.openstack:FaultWrapper.factory
    
    [filter:noauth]
    paste.filter_factory = nova.api.openstack.auth:NoAuthMiddleware.factory
    
    [filter:noauth_v3]
    paste.filter_factory = nova.api.openstack.auth:NoAuthMiddlewareV3.factory
    
    [filter:ratelimit]
    paste.filter_factory = nova.api.openstack.compute.limits:RateLimitingMiddleware.factory
    
    [filter:sizelimit]
    paste.filter_factory = nova.api.sizelimit:RequestBodySizeLimiter.factory
    
    [app:osapi_compute_app_v2]
    paste.app_factory = nova.api.openstack.compute:APIRouter.factory
    
    [app:osapi_compute_app_v3]
    paste.app_factory = nova.api.openstack.compute:APIRouterV3.factory
    
    [pipeline:oscomputeversions]
    pipeline = faultwrap oscomputeversionapp
    
    [app:oscomputeversionapp]
    paste.app_factory = nova.api.openstack.compute.versions:Versions.factory
    
    ##########
    # Shared #
    ##########
    
    [filter:keystonecontext]
    paste.filter_factory = nova.api.auth:NovaKeystoneContext.factory
    
    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory

我们在这里只关注OpenStack的app：

    #############
    # OpenStack #
    #############
    
    [composite:osapi_compute]
    use = call:nova.api.openstack.urlmap:urlmap_factory
    /: oscomputeversions
    /v1.1: openstack_compute_api_v2
    /v2: openstack_compute_api_v2
    /v3: openstack_compute_api_v3
    
    [composite:openstack_compute_api_v2]
    use = call:nova.api.auth:pipeline_factory
    noauth = faultwrap sizelimit noauth ratelimit osapi_compute_app_v2
    keystone = faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
    keystone_nolimit = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v2
    
    [composite:openstack_compute_api_v3]
    use = call:nova.api.auth:pipeline_factory_v3
    noauth = faultwrap sizelimit noauth_v3 osapi_compute_app_v3
    keystone = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v3
    
    [filter:faultwrap]
    paste.filter_factory = nova.api.openstack:FaultWrapper.factory
    
    [filter:noauth]
    paste.filter_factory = nova.api.openstack.auth:NoAuthMiddleware.factory
    
    [filter:noauth_v3]
    paste.filter_factory = nova.api.openstack.auth:NoAuthMiddlewareV3.factory
    
    [filter:ratelimit]
    paste.filter_factory = nova.api.openstack.compute.limits:RateLimitingMiddleware.factory
    
    [filter:sizelimit]
    paste.filter_factory = nova.api.sizelimit:RequestBodySizeLimiter.factory
    
    [app:osapi_compute_app_v2]
    paste.app_factory = nova.api.openstack.compute:APIRouter.factory
    
    [app:osapi_compute_app_v3]
    paste.app_factory = nova.api.openstack.compute:APIRouterV3.factory
    
    [pipeline:oscomputeversions]
    pipeline = faultwrap oscomputeversionapp
    
    [app:oscomputeversionapp]
    paste.app_factory = nova.api.openstack.compute.versions:Versions.factory
    
    ##########
    # Shared #
    ##########
    
    [filter:keystonecontext]
    paste.filter_factory = nova.api.auth:NovaKeystoneContext.factory
    
    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory

这个paste.ini是不是看着很眼熟，里面就是一些composite、app、filter和pipeline。OpenStack的nova-api就是根据这个配置来启动一个个WSGI service的。  

未完待续  

[1]: http://pythonpaste.org/deploy/
