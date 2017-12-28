---
layout: post
title: "openstack icehouse new future"
description: ""
category: articles
tags: []
---
# OpenStack Compute(Nova)

## Compute Drivers
---------------

### Libvirt(KVM)

* 支持传递修改的kernel参数给启动的虚拟实例。kernel参数从储存在glance的image metadata的os_command_line中获取。
* 支持使用VirtIO SCSI为实例提供块设备的驱动，替代之前的VirtIO Block。
* 支持给实例添加VirtIO RNG device。VirtIO RNG 是半虚拟化的随机数产生器，它可以给虚拟实例提供熵池。当然，也可以为主机添加物理的随机数设备。
* 支持配置实例使用视频驱动。
* 支持watchdog。使用i6300esb作为watch设备。可以通过设置image属性中的hw_watchdog_action或flavor中的属性来启用。支持的hw_watchdog_action属性包括poweroff、reset、pause和none，可以在实例运行失败的时候被触发执行。
* 禁用HPET。
* 支持在创建启动实例时等待接受neutron的事件，这样可以提供更好的可靠性。
