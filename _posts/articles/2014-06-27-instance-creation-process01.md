---
layout: post
title: "OpenStack创建虚拟机流程简述01"
description: ""
category: OpenStack 
tags: [nova]
---
在OpenStack中，大部分操作是围绕虚拟机展开的。而在虚拟机中，最核心的操作恐怕非创建虚拟机莫属。下面，我就简单分析下OpenStack中创建虚拟机的流程。
首先，根据前面几篇文章可知，nova（其他OpenStack组件均如此）对外暴露的功能是通过HTTP RESTful api来实现的。用户在dashboard中触发了一个创建虚拟机的请求，这个请求最终会由openstack-nova-api接收，根据路由规则，会映射到nova/api/openstack/compute/servers.py中Controller的create()：

    @wsgi.response(202)
    @wsgi.serializers(xml=FullServerTemplate)
    @wsgi.deserializers(xml=CreateDeserializer)
    def create(self, req, body):
        ...
        try:
            _get_inst_type = flavors.get_flavor_by_flavor_id
            inst_type = _get_inst_type(flavor_id, ctxt=context,
                                       read_deleted="no")

            (instances, resv_id) = self.compute_api.create(context,
                            inst_type,
                            image_uuid,
                            display_name=name,
                            display_description=name,
                            key_name=key_name, 
                            metadata=server_dict.get('metadata', {}),
                            access_ip_v4=access_ip_v4,
                            access_ip_v6=access_ip_v6,
                            injected_files=injected_files,
                            admin_password=password,
                            min_count=min_count,
                            max_count=max_count,
                            requested_networks=requested_networks,
                            security_group=sg_names,
                            user_data=user_data,
                            availability_zone=availability_zone,
                            config_drive=config_drive,
                            block_device_mapping=block_device_mapping,
                            auto_disk_config=auto_disk_config,
                            scheduler_hints=scheduler_hints,
                            legacy_bdm=legacy_bdm)
        ...

这个函数的前半部分基本是对输入req参数的提取、封装及验证，最后会调用self.compute\_api.create()来执行操作。self.compute\_api默认是nova.compute.api.API：

    class API(base.Base):
        """API for interacting with the compute manager."""

        def __init__(self, image_service=None, network_api=None, volume_api=None,
                     security_group_api=None, **kwargs):
            self.image_service = (image_service or
                                  glance.get_default_image_service())
    
            self.network_api = network_api or network.API()
            self.volume_api = volume_api or volume.API()
            self.security_group_api = (security_group_api or
                openstack_driver.get_openstack_security_group_driver())
            self.consoleauth_rpcapi = consoleauth_rpcapi.ConsoleAuthAPI()
            self.compute_rpcapi = compute_rpcapi.ComputeAPI()
            self._compute_task_api = None
            self.servicegroup_api = servicegroup.API()
            self.notifier = rpc.get_notifier('compute', CONF.host)
    
            super(API, self).__init__(**kwargs)
            
        ...
        
self.compute\_api.create()实际上就是调用的nova.compute.api.API.create()

        @hooks.add_hook("create_instance")
        def create(self, context, instance_type,
                   image_href, kernel_id=None, ramdisk_id=None,
                   min_count=None, max_count=None,
                   display_name=None, display_description=None,
                   key_name=None, key_data=None, security_group=None,
                   availability_zone=None, user_data=None, metadata=None,
                   injected_files=None, admin_password=None,
                   block_device_mapping=None, access_ip_v4=None,
                   access_ip_v6=None, requested_networks=None, config_drive=None,
                   auto_disk_config=None, scheduler_hints=None, legacy_bdm=True):
            """Provision instances, sending instance information to the
            scheduler.  The scheduler will determine where the instance(s)
            go and will handle creating the DB entries.
    
            Returns a tuple of (instances, reservation_id)
            """
    
            self._check_create_policies(context, availability_zone,
                    requested_networks, block_device_mapping)
    
            if requested_networks and max_count > 1 and utils.is_neutron():
                self._check_multiple_instances_neutron_ports(requested_networks)
    
            return self._create_instance(
                                   context, instance_type,
                                   image_href, kernel_id, ramdisk_id,
                                   min_count, max_count,
                                   display_name, display_description,
                                   key_name, key_data, security_group,
                                   availability_zone, user_data, metadata,
                                   injected_files, admin_password,
                                   access_ip_v4, access_ip_v6,
                                   requested_networks, config_drive,
                                   block_device_mapping, auto_disk_config,
                                   scheduler_hints=scheduler_hints,
                                   legacy_bdm=legacy_bdm)
                                   
最后，实际上是调用的nova.compute.api.API的私有函数\_create\_instance()：

    def _create_instance(self, context, instance_type,
               image_href, kernel_id, ramdisk_id,
               min_count, max_count,
               display_name, display_description,
               key_name, key_data, security_groups,
               availability_zone, user_data, metadata,
               injected_files, admin_password,
               access_ip_v4, access_ip_v6,
               requested_networks, config_drive,
               block_device_mapping, auto_disk_config,
               reservation_id=None, scheduler_hints=None,
               legacy_bdm=True):
        ...

        self.compute_task_api.build_instances(context,
                instances=instances, image=boot_meta,
                filter_properties=filter_properties,
                admin_password=admin_password,
                injected_files=injected_files,
                requested_networks=requested_networks,
                security_groups=security_groups,
                block_device_mapping=block_device_mapping,
                legacy_bdm=False)

        return (instances, reservation_id)
        
这个函数在前面主要对block device、image、security\_groups和availability\_zone进行一些处理，最后会调用self.compute\_task\_api.build\_instances()。self.compute\_task\_api默认是nova.conductor.ComputeTask.API：

    def ComputeTaskAPI(*args, **kwargs):
        use_local = kwargs.pop('use_local', False)
        if oslo.config.cfg.CONF.conductor.use_local or use_local:
            api = conductor_api.LocalComputeTaskAPI
        else:
            api = conductor_api.ComputeTaskAPI
        return api(*args, **kwargs)

它默认返回的是nova.conductor.api.ComputeTaskAPI，看它的build\_instances()函数：

    def build_instances(self, context, instances, image, filter_properties,
            admin_password, injected_files, requested_networks,
            security_groups, block_device_mapping, legacy_bdm=True):
        self.conductor_compute_rpcapi.build_instances(context,
                instances=instances, image=image,
                filter_properties=filter_properties,
                admin_password=admin_password, injected_files=injected_files,
                requested_networks=requested_networks,
                security_groups=security_groups,
                block_device_mapping=block_device_mapping,
                legacy_bdm=legacy_bdm)
                
self.conductor\_compute\_rpcapi是nova.conductor.rpcapi.ComputeTaskAPI，它其实就是对rpc call或cast的封装，用来通过消息队列远程调用相应manager里的函数，这里它调用的是nova.conductor.manager.ComputeTaskManager.build_instances()：

    def build_instances(self, context, instances, image, filter_properties,
            admin_password, injected_files, requested_networks,
            security_groups, block_device_mapping, legacy_bdm=True):
        request_spec = scheduler_utils.build_request_spec(context, image,
                                                          instances)
        # NOTE(alaski): For compatibility until a new scheduler method is used.
        request_spec.update({'block_device_mapping': block_device_mapping,
                             'security_group': security_groups})
        self.scheduler_rpcapi.run_instance(context, request_spec=request_spec,
                admin_password=admin_password, injected_files=injected_files,
                requested_networks=requested_networks, is_first_time=True,
                filter_properties=filter_properties,
                legacy_bdm_in_spec=legacy_bdm)
                
这里调用的是self.scheduler\_rpcapi.run\_instance()去执行调度操作，对应的远程调用函数在nova.scheduler.manager.SchedulerManager.run\_instance()中：

    def run_instance(self, context, request_spec, admin_password,
            injected_files, requested_networks, is_first_time,
            filter_properties, legacy_bdm_in_spec=True):
        """Tries to call schedule_run_instance on the driver.
        Sets instance vm_state to ERROR on exceptions
        """
        instance_uuids = request_spec['instance_uuids']
        with compute_utils.EventReporter(context, conductor_api.LocalAPI(),
                                         'schedule', *instance_uuids):
            try:
                return self.driver.schedule_run_instance(context,
                        request_spec, admin_password, injected_files,
                        requested_networks, is_first_time, filter_properties,
                        legacy_bdm_in_spec)

            except exception.NoValidHost as ex:
                # don't re-raise
                self._set_vm_state_and_notify('run_instance',
                                              {'vm_state': vm_states.ERROR,
                                              'task_state': None},
                                              context, ex, request_spec)
            except Exception as ex:
                with excutils.save_and_reraise_exception():
                    self._set_vm_state_and_notify('run_instance',
                                                  {'vm_state': vm_states.ERROR,
                                                  'task_state': None},
                                                  context, ex, request_spec)
                                                  
默认的scheduler\_driver是nova.scheduler.filter\_scheduler.FilterScheduler，它会根据在nova.conf中配置的filter执行过滤，并使用配置weight的去计算权重并排序：

    def schedule_run_instance(self, context, request_spec,
                              admin_password, injected_files,
                              requested_networks, is_first_time,
                              filter_properties, legacy_bdm_in_spec):
        ...

        weighed_hosts = self._schedule(context, request_spec,
                                       filter_properties, instance_uuids)

        # NOTE: Pop instance_uuids as individual creates do not need the
        # set of uuids. Do not pop before here as the upper exception
        # handler fo NoValidHost needs the uuid to set error state
        instance_uuids = request_spec.pop('instance_uuids')

        # NOTE(comstud): Make sure we do not pass this through.  It
        # contains an instance of RpcContext that cannot be serialized.
        filter_properties.pop('context', None)

        for num, instance_uuid in enumerate(instance_uuids):
            request_spec['instance_properties']['launch_index'] = num

            try:
                ...

                self._provision_resource(context, weighed_host,
                                         request_spec,
                                         filter_properties,
                                         requested_networks,
                                         injected_files, admin_password,
                                         is_first_time,
                                         instance_uuid=instance_uuid,
                                         legacy_bdm_in_spec=legacy_bdm_in_spec)
            except Exception as ex:
                # NOTE(vish): we don't reraise the exception here to make sure
                #             that all instances in the request get set to
                #             error properly
                driver.handle_schedule_error(context, ex, instance_uuid,
                                             request_spec)
            # scrub retry host list in case we're scheduling multiple
            # instances:
            retry = filter_properties.get('retry', {})
            retry['hosts'] = []

        self.notifier.info(context, 'scheduler.run_instance.end', payload)

self.\_schedule()返回的是经过排序的hosts，然后循环需要创建的instance\_uuids，最后调用self.\_provision_resource()：

    def _provision_resource(self, context, weighed_host, request_spec,
            filter_properties, requested_networks, injected_files,
            admin_password, is_first_time, instance_uuid=None,
            legacy_bdm_in_spec=True):
            
        ...
        
        try:
            updated_instance = driver.instance_update_db(context,
                                                         instance_uuid)
        except exception.InstanceNotFound:
            LOG.warning(_("Instance disappeared during scheduling"),
                        context=context, instance_uuid=instance_uuid)

        else:
            scheduler_utils.populate_filter_properties(filter_properties,
                    weighed_host.obj)

            self.compute_rpcapi.run_instance(context,
                    instance=updated_instance,
                    host=weighed_host.obj.host,
                    request_spec=request_spec,
                    filter_properties=filter_properties,
                    requested_networks=requested_networks,
                    injected_files=injected_files,
                    admin_password=admin_password, is_first_time=is_first_time,
                    node=weighed_host.obj.nodename,
                    legacy_bdm_in_spec=legacy_bdm_in_spec)
                    
在这里调用self.compute\_rpcapi.run\_instance()，根据名字就可以判断，它会远程调用nova.compute.manager.ComputeManager.run\_instance()。消息队列会根据传入的host作为topic，把消息路由到相应的nova-compute所在的节点。这个节点收到消息后，会去执行相应的函数调用。

未完待续
