---
layout: post
title: "OpenStack创建虚拟机流程简述02"
description: ""
category: OpenStack 
tags: [nova]
---
在经过nova-scheduler的过滤和排序后，请求会发送到相应host的nova-compute中，调用nova.compute.manager.ComputeManager.run\_instance()：

    @wrap_exception()
    @reverts_task_state
    @wrap_instance_event
    @wrap_instance_fault
    def run_instance(self, context, instance, request_spec,
                     filter_properties, requested_networks,
                     injected_files, admin_password,
                     is_first_time, node, legacy_bdm_in_spec):

        if filter_properties is None:
            filter_properties = {}

        @utils.synchronized(instance['uuid'])
        def do_run_instance():
            self._run_instance(context, request_spec,
                    filter_properties, requested_networks, injected_files,
                    admin_password, is_first_time, node, instance,
                    legacy_bdm_in_spec)
        do_run_instance()
        
最后会调用\_run\_instance()：

    def _run_instance(self, context, request_spec,
                      filter_properties, requested_networks, injected_files,
                      admin_password, is_first_time, node, instance,
                      legacy_bdm_in_spec):
        """Launch a new instance with specified options."""

        extra_usage_info = {}

        def notify(status, msg="", fault=None, **kwargs):
            """Send a create.{start,error,end} notification."""
            type_ = "create.%(status)s" % dict(status=status)
            info = extra_usage_info.copy()
            info['message'] = unicode(msg)
            self._notify_about_instance_usage(context, instance, type_,
                    extra_usage_info=info, fault=fault, **kwargs)

        try:
            self._prebuild_instance(context, instance)

            if request_spec and request_spec.get('image'):
                image_meta = request_spec['image']
            else:
                image_meta = {}

            extra_usage_info = {"image_name": image_meta.get('name', '')}

            notify("start")  # notify that build is starting

            instance, network_info = self._build_instance(context,
                    request_spec, filter_properties, requested_networks,
                    injected_files, admin_password, is_first_time, node,
                    instance, image_meta, legacy_bdm_in_spec)
            notify("end", msg=_("Success"), network_info=network_info)
            
self.\_build\_instance()最终执行创建虚拟机的操作：

    def _build_instance(self, context, request_spec, filter_properties,
            requested_networks, injected_files, admin_password, is_first_time,
            node, instance, image_meta, legacy_bdm_in_spec):
            
        ...
        
        try:
        
            ...
            
            with rt.instance_claim(context, instance, limits):
            
                ...

                network_info = self._allocate_network(context, instance,
                        requested_networks, macs, security_groups,
                        dhcp_options)

                ...

                instance = self._spawn(context, instance, image_meta,
                                       network_info, block_device_info,
                                       injected_files, admin_password,
                                       set_access_ip=set_access_ip)
                                       
        ...

        # spawn success
        return instance, network_info
        
着重看self.\_allocate\_network()和self.\_spawn()这两个函数。self.\_allocate\_network()主要任务是调用network-api（我们使用的是neutron）在选择的network上创建相应的网络设备。self.\_spawn()是调用相应的driver去创建虚拟机。我们在这里只关注\_spawn()：

    @object_compat
    def _spawn(self, context, instance, image_meta, network_info,
               block_device_info, injected_files, admin_password,
               set_access_ip=False):
        """Spawn an instance with error logging and update its power state."""
        instance.vm_state = vm_states.BUILDING
        instance.task_state = task_states.SPAWNING
        instance.save(expected_task_state=task_states.BLOCK_DEVICE_MAPPING)

        try:
            self.driver.spawn(context, instance, image_meta,
                              injected_files, admin_password,
                              network_info,
                              block_device_info)
        except Exception:
            with excutils.save_and_reraise_exception():
                LOG.exception(_('Instance failed to spawn'), instance=instance)

        current_power_state = self._get_power_state(context, instance)

        instance.power_state = current_power_state
        instance.vm_state = vm_states.ACTIVE
        instance.task_state = None
        instance.launched_at = timeutils.utcnow()
        
函数前半部分和后半部分主要是更新一些虚拟机的状态，self.driver.spawn()会调用相应的driver，我们这里使用的是nova.virt.libvirt.LibvirtDriver，看它的spawn()：

    def spawn(self, context, instance, image_meta, injected_files,
              admin_password, network_info=None, block_device_info=None):
              
        ...
        
        self._create_image(context, instance,
                           disk_info['mapping'],
                           network_info=network_info,
                           block_device_info=block_device_info,
                           files=injected_files,
                           admin_pass=admin_password)
        xml = self.to_xml(context, instance, network_info,
                          disk_info, image_meta,
                          block_device_info=block_device_info,
                          write_to_disk=True)

        self._create_domain_and_network(context, xml, instance, network_info,
                                        block_device_info)
        ...
        
self.\_create\_image()是获取虚拟机的镜像，self.to\_xml()是根据instance信息生成了一个名为libvirt.xml的libvirt domain配置，然后调用self.\_create\_domain\_and\_network()：

    def _create_domain_and_network(self, context, xml, instance, network_info,
                                   block_device_info=None, power_on=True,
                                   reboot=False, vifs_already_plugged=False):

        ...
        
        try:
            with self.virtapi.wait_for_instance_event(
                    instance, events, deadline=timeout,
                    error_callback=self._neutron_failed_callback):
                self.plug_vifs(instance, network_info)
                self.firewall_driver.setup_basic_filtering(instance,
                                                           network_info)
                self.firewall_driver.prepare_instance_filter(instance,
                                                             network_info)
                domain = self._create_domain(
                    xml, instance=instance,
                    launch_flags=launch_flags,
                    power_on=power_on)

                self.firewall_driver.apply_instance_filter(instance,
                                                           network_info)
                                                           
        ...
        
        return domain
        
因为之前\_allocate\_network()已经为虚拟机创建了相应的port，self.plug\_vifs()就负责将这些port和虚拟机的网络接口连接起来。假设network使用的是neutron，driver使用的是openvswitch，那么这个操作其实就是使用ovs-vsctl add-port为虚拟机添加ovs port。如果在这里还为虚拟机添加了security group的话，那么还需要在这个port上添加linux bridge和veth pair，将虚拟机的tap和ovs port连接起来（因为ovs port无法进行iptables过滤）：

![enter image description here][1]

port创建好之后，通过self.firewall_driver根据安全组为虚拟机创建访问规则。
self.\_create\_domain()是我们这篇文章的主角，它通过libvirt driver的connection调用hypervisor去执行相关的虚拟机创建和启动等操作：

    def _create_domain(self, xml=None, domain=None,
                       instance=None, launch_flags=0, power_on=True):
                       
        ...

        if xml:
            try:
                domain = self._conn.defineXML(xml)
            except Exception as e:
                LOG.error(_("An error occurred while trying to define a domain"
                            " with xml: %s") % xml)
                raise e

        if power_on:
            try:
                domain.createWithFlags(launch_flags)
            except Exception as e:
                with excutils.save_and_reraise_exception():
                    LOG.error(_("An error occurred while trying to launch a "
                                "defined domain with xml: %s") %
                              domain.XMLDesc(0))

        ...

        return domain
        
self.\_conn.defineXML()是根据xml定义一个 domain，其实就相当于：

    virsh define libvirt.xml
    
最后，如果启动的话，就使用domain.createWithFlags()去启动一个虚拟机，在这里它是调用的libvirt python binding interface去和libvirt通信。关于libvirt python binding，可以看[这里][2]。

这样的话，一个虚拟机的核心创建流程就结束了。


  [1]: http://docs.openstack.org/grizzly/openstack-network/admin/content/figures/2/figures/under-the-hood-scenario-1-ovs-compute.png
  [2]: http://libvirt.org/python.html
