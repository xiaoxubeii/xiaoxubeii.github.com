---
layout: post
title: "nova compute创建虚拟机分配network"
description: ""
category: articles
tags: [Nova, Network]
---
## nova-compute创建虚拟机分配network过程 ##
在/nova/compute/manager.py中：

    def _build_instance(self, context, request_spec, filter_properties,
            requested_networks, injected_files, admin_password, is_first_time,
            node, instance, image_meta, legacy_bdm_in_spec):
        context = context.elevated()

        ...
        
        try:
            limits = filter_properties.get('limits', {})
            with rt.instance_claim(context, instance, limits):
                macs = self.driver.macs_for_instance(instance)
                dhcp_options = self.driver.dhcp_options_for_instance(instance)

                network_info = self._allocate_network(context, instance,
                        requested_networks, macs, security_groups,
                        dhcp_options)
                        
        ...
        
        return instance, network_info

通过_allocate_network()，创建必要的port和security group。

        def _allocate_network(self, context, instance, requested_networks, macs,
                          security_groups, dhcp_options):
        """Start network allocation asynchronously.  Return an instance
        of NetworkInfoAsyncWrapper that can be used to retrieve the
        allocated networks when the operation has finished.
        """
        # NOTE(comstud): Since we're allocating networks asynchronously,
        # this task state has little meaning, as we won't be in this
        # state for very long.
        instance = self._instance_update(context, instance['uuid'],
                                         vm_state=vm_states.BUILDING,
                                         task_state=task_states.NETWORKING,
                                         expected_task_state=[None])
        is_vpn = pipelib.is_vpn_image(instance['image_ref'])
        return network_model.NetworkInfoAsyncWrapper(
                self._allocate_network_async, context, instance,
                requested_networks, macs, security_groups, is_vpn,
                dhcp_options)
                
network_model.NetworkInfoAsyncWrapper()创建一个异步的wrapper，返回执行_allocate_network_async()的结果。

    def _allocate_network_async(self, context, instance, requested_networks,
                                macs, security_groups, is_vpn, dhcp_options):
                                
        ...
        
        for attempt in range(1, attempts + 1):
            try:
                nwinfo = self.network_api.allocate_for_instance(
                        context, instance, vpn=is_vpn,
                        requested_networks=requested_networks,
                        macs=macs,
                        security_groups=security_groups,
                        dhcp_options=dhcp_options)
                        
        ...
最后通过network_api.allocate_for_instance()里执行操作。
allocate_for_instance()在/nova/network/neutronv2/api.py中：

    def allocate_for_instance(self, context, instance, **kwargs):
    
        ...
        
        if requested_networks:
            for network_id, fixed_ip, port_id in requested_networks:
                if port_id:
                    port = neutron.show_port(port_id)['port']
                    if port.get('device_id'):
                        raise exception.PortInUse(port_id=port_id)
                    if hypervisor_macs is not None:
                        if port['mac_address'] not in hypervisor_macs:
                            raise exception.PortNotUsable(port_id=port_id,
                                instance=instance['display_name'])
                        else:
                            available_macs.discard(port['mac_address'])
                    network_id = port['network_id']
                    ports[network_id] = port
                elif fixed_ip and network_id:
                    fixed_ips[network_id] = fixed_ip
                if network_id:
                    net_ids.append(network_id)

        ...
        
        touched_port_ids = []
        created_port_ids = []
        for network in nets:
            if (security_groups and not (
                    network['subnets']
                    and network.get('port_security_enabled', True))):

                raise exception.SecurityGroupCannotBeApplied()
            network_id = network['id']
            zone = 'compute:%s' % instance['availability_zone']
            port_req_body = {'port': {'device_id': instance['uuid'],
                                      'device_owner': zone}}
            try:
                port = ports.get(network_id)
                self._populate_neutron_extension_values(instance,
                                                        port_req_body)
                # Requires admin creds to set port bindings
                port_client = (neutron if not
                               self._has_port_binding_extension() else
                               neutronv2.get_client(context, admin=True))
                if port:
                    port_client.update_port(port['id'], port_req_body)
                    touched_port_ids.append(port['id'])
                else:
                    //创建port
                    created_port_ids.append(self._create_port(
                            port_client, instance, network_id,
                            port_req_body, fixed_ips.get(network_id),
                            security_group_ids, available_macs, dhcp_opts))
           
           ...

        nw_info = self.get_instance_nw_info(context, instance, networks=nets)
        return network_model.NetworkInfo([port for port in nw_info
                                          if port['id'] in created_port_ids +
                                                           touched_port_ids])
