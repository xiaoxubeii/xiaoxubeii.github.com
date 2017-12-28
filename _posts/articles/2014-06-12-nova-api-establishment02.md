---
layout: post
title: "nova-api建立流程02"
description: ""
category: articles
tags: [nova, nova-api]
---
# Compute app启动
在api-paste.ini中有：

    ...
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
    ...
    [app:osapi_compute_app_v2]
    paste.app_factory = nova.api.openstack.compute:APIRouter.factory
    
composite:osapi_compute是一个urlmap的composite section，它会根据不同的url映射到不同的app中。比如，url为/v2，那么openstack\_compute\_api\_v2这个composite app就会启动：

    [composite:openstack_compute_api_v2]
    use = call:nova.api.auth:pipeline_factory
    noauth = faultwrap sizelimit noauth ratelimit osapi_compute_app_v2
    keystone = faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
    keystone_nolimit = faultwrap sizelimit authtoken keystonecontext osapi_compute_app_v2
    
openstack\_compute\_api\_v2这个composite section会调用nova.api.auth:pipeline_factory来返回一个app：

    def _load_pipeline(loader, pipeline):
        filters = [loader.get_filter(n) for n in pipeline[:-1]]
        app = loader.get_app(pipeline[-1])
        filters.reverse()
        for filter in filters:
            app = filter(app)
        return app


    def pipeline_factory(loader, global_conf, **local_conf):
        """A paste pipeline replica that keys off of auth_strategy."""
        pipeline = local_conf[CONF.auth_strategy]
        if not CONF.api_rate_limit:
            limit_name = CONF.auth_strategy + '_nolimit'
            pipeline = local_conf.get(limit_name, pipeline)
        pipeline = pipeline.split()
        # NOTE (Alex Xu): This is just for configuration file compatibility.
        # If the configuration file still contains 'ratelimit_v3', just ignore it.
        # We will remove this code at next release (J)
        if 'ratelimit_v3' in pipeline:
            LOG.warn(_('ratelimit_v3 is removed from v3 api.'))
            pipeline.remove('ratelimit_v3')
        return _load_pipeline(loader, pipeline)

pipeline_factory会根据auth\_strategy这个配置选择相应的pipe filter链，比如这里的配置是keystone，那么执行的是：

    keystone = faultwrap sizelimit authtoken keystonecontext ratelimit osapi_compute_app_v2
    
这个pipeline前面的都是filter，比如faultwrap、sizelimit、authtoken等，最后一个是app：

    [app:osapi_compute_app_v2]
    paste.app_factory = nova.api.openstack.compute:APIRouter.factory
    
这里会调用nova.api.openstack.compute里的APIRouter.factory去返回一个app：
    
    # cat /nova/api/openstack/compute/__init__.py

    class APIRouter(nova.api.openstack.APIRouter):
        """Routes requests on the OpenStack API to the appropriate controller
        and method.
        """
        ExtensionManager = extensions.ExtensionManager
    
        def _setup_routes(self, mapper, ext_mgr, init_only):
            ...
    
    # cat /nova/api/openstack/__init__.py
            
    class APIRouter(base_wsgi.Router):
        """Routes requests on the OpenStack API to the appropriate controller
        and method.
        """
        ExtensionManager = None  # override in subclasses
    
        @classmethod
        def factory(cls, global_config, **local_config):
            """Simple paste factory, :class:`nova.wsgi.Router` doesn't have one."""
            return cls()
    
        def __init__(self, ext_mgr=None, init_only=None):
            if ext_mgr is None:
                if self.ExtensionManager:
                    ext_mgr = self.ExtensionManager()
                else:
                    raise Exception(_("Must specify an ExtensionManager class"))
    
            mapper = ProjectMapper()
            self.resources = {}
            self._setup_routes(mapper, ext_mgr, init_only)
            self._setup_ext_routes(mapper, ext_mgr, init_only)
            self._setup_extensions(ext_mgr)
            super(APIRouter, self).__init__(mapper)
    
        def _setup_ext_routes(self, mapper, ext_mgr, init_only):
            ...
    
        def _setup_extensions(self, ext_mgr):
            ...
    
        def _setup_routes(self, mapper, ext_mgr, init_only):
            raise NotImplementedError()
            
着重看nova.api.openstack.APIRouter的初始化过程。sefl.\_setup\_routes()会创建核心资源的路由，它的实现代码在子类nova.api.openstack.compute.APIRouter中：

    def _setup_routes(self, mapper, ext_mgr, init_only):
        if init_only is None or 'versions' in init_only:
            self.resources['versions'] = versions.create_resource()
            mapper.connect("versions", "/",
                        controller=self.resources['versions'],
                        action='show',
                        conditions={"method": ['GET']})

        mapper.redirect("", "/")

        if init_only is None or 'consoles' in init_only:
            self.resources['consoles'] = consoles.create_resource()
            mapper.resource("console", "consoles",
                        controller=self.resources['consoles'],
                        parent_resource=dict(member_name='server',
                        collection_name='servers'))

        if init_only is None or 'consoles' in init_only or \
                'servers' in init_only or 'ips' in init_only:
            self.resources['servers'] = servers.create_resource(ext_mgr)
            mapper.resource("server", "servers",
                            controller=self.resources['servers'],
                            collection={'detail': 'GET'},
                            member={'action': 'POST'})

        if init_only is None or 'ips' in init_only:
            self.resources['ips'] = ips.create_resource()
            mapper.resource("ip", "ips", controller=self.resources['ips'],
                            parent_resource=dict(member_name='server',
                                                 collection_name='servers'))

        if init_only is None or 'images' in init_only:
            self.resources['images'] = images.create_resource()
            mapper.resource("image", "images",
                            controller=self.resources['images'],
                            collection={'detail': 'GET'})

        if init_only is None or 'limits' in init_only:
            self.resources['limits'] = limits.create_resource()
            mapper.resource("limit", "limits",
                            controller=self.resources['limits'])

        if init_only is None or 'flavors' in init_only:
            self.resources['flavors'] = flavors.create_resource()
            mapper.resource("flavor", "flavors",
                            controller=self.resources['flavors'],
                            collection={'detail': 'GET'},
                            member={'action': 'POST'})

        if init_only is None or 'image_metadata' in init_only:
            self.resources['image_metadata'] = image_metadata.create_resource()
            image_metadata_controller = self.resources['image_metadata']

            mapper.resource("image_meta", "metadata",
                            controller=image_metadata_controller,
                            parent_resource=dict(member_name='image',
                            collection_name='images'))

            mapper.connect("metadata",
                           "/{project_id}/images/{image_id}/metadata",
                           controller=image_metadata_controller,
                           action='update_all',
                           conditions={"method": ['PUT']})

        if init_only is None or 'server_metadata' in init_only:
            self.resources['server_metadata'] = \
                server_metadata.create_resource()
            server_metadata_controller = self.resources['server_metadata']

            mapper.resource("server_meta", "metadata",
                            controller=server_metadata_controller,
                            parent_resource=dict(member_name='server',
                            collection_name='servers'))

            mapper.connect("metadata",
                           "/{project_id}/servers/{server_id}/metadata",
                           controller=server_metadata_controller,
                           action='update_all',
                           conditions={"method": ['PUT']})

它会创建诸如servers、images、flavors等compute的核心资源，然后通过mapper.resource()创建相应的RESTful api接口。
self.\_setup\_ext\_routes()和self.\_setup\_extensions()会加载extension并创建路由。extension扩展资源其实分两类，一类本身就是资源，只不过不是核心。另一类是对核心资源的扩展。self.\_setup\_ext\_routes()是加载和创建扩展资源的路由，其实就是对新资源的mapper.resource()或mapper.connect()。self.\_setup\_extensions()是加载核心资源的扩展，这里面是利用metaclass向原有的controller进行注入，具体的执行过程我会单独用一篇文章介绍。
这样的话，整个openstack nova api的route就建立好了。用户在输入url的时候，首先会通过urlmap映射到相应的app中，经过pipeline进行filter，最后调用相应的APIRouter，APIRouter的最终父类在/nova/wsgi.py的Router中：

    class Router(object):
        """WSGI middleware that maps incoming requests to WSGI apps."""
    
        def __init__(self, mapper):
            """Create a router for the given routes.Mapper.
    
            Each route in `mapper` must specify a 'controller', which is a
            WSGI app to call.  You'll probably want to specify an 'action' as
            well and have your controller be an object that can route
            the request to the action-specific method.
    
            Examples:
              mapper = routes.Mapper()
              sc = ServerController()
    
              # Explicit mapping of one route to a controller+action
              mapper.connect(None, '/svrlist', controller=sc, action='list')
    
              # Actions are all implicitly defined
              mapper.resource('server', 'servers', controller=sc)
    
              # Pointing to an arbitrary WSGI app.  You can specify the
              # {path_info:.*} parameter so the target app can be handed just that
              # section of the URL.
              mapper.connect(None, '/v1.0/{path_info:.*}', controller=BlogApp())
    
            """
            self.map = mapper
            self._router = routes.middleware.RoutesMiddleware(self._dispatch,
                                                              self.map)
    
        @webob.dec.wsgify(RequestClass=Request)
        def __call__(self, req):
            """Route the incoming request to a controller based on self.map.
    
            If no match, return a 404.
    
            """
            return self._router
    
        @staticmethod
        @webob.dec.wsgify(RequestClass=Request)
        def _dispatch(req):
            """Dispatch the request to the appropriate controller.
    
            Called by self._router after matching the incoming request to a route
            and putting the information into req.environ.  Either returns 404
            or the routed WSGI app's response.
    
            """
            match = req.environ['wsgiorg.routing_args'][1]
            if not match:
                return webob.exc.HTTPNotFound()
            app = match['controller']
            return app

它其实就是一个可以callable的类，webob会对request进行封装，然后在每次调用时返回一个self.\_router，self.\_router在调用的时候会依据路由信息通过\_dispatch()将request分发到相应controller的action中，这样一个完整的url调用就结束了。

未完待续


