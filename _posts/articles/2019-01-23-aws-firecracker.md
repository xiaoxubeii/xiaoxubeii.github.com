---
layout: post
title: "Introduction for AWS Firecracker"
description: ""
category: articles
tags: [AWS Firecracker, MicroVM]
---

# AWS Firecracker 和 KataContainer 初探（一）
## AWS Firecracker
首先贴两段官网对 Firecracker 的定义：
> Firecracker is an open source virtualization technology that is purpose-built for creating and managing secure, multi-tenant container and function-based services that provide serverless operational models. Firecracker runs workloads in lightweight virtual machines, called microVMs, which combine the security and isolation properties provided by hardware virtualization technology with the speed and flexibility of containers.

> The main component of Firecracker is a virtual machine monitor (VMM) that uses the Linux Kernel Virtual Machine (KVM) to create and run microVMs. Firecracker has a minimalist design. It excludes unnecessary devices and guest-facing functionality to reduce the memory footprint and attack surface area of each microVM. This improves security, decreases the startup time, and increases hardware utilization. Firecracker currently supports Intel CPUs, with planned AMD and Arm support. Firecracker will also be integrated with popular container runtimes.

实际上 Firecracker 就是一个 VMM，可以运行轻量级的虚拟机，也就是所谓的 **microVM**。Firecracker 底层还是使用 KVM 做 CPU 和内存的硬件虚拟化，然后配合 KVM 实现其他设备的虚拟化。它对标的是 qemu，更确切的说是魔改的 qemu-lite。因为去除了很多累赘设备和做了优化，配合精简的 Linux Kernel，可以将虚拟机启动时间缩短到秒级，甚至毫秒级。Firecracker 是 AWS Lambda 和 AWS Fargate 的后端支持，背景可见一斑。不过众所周知，开源代码和内部生产必然有很大差距，大家秉承 PoC 的精神即可 ：）。

### 如何使用
由于 Firecracker 还处于 0.1x 阶段，在使用上还是非常简单和技术化。
运行 Firecracker 有两个先决条件：

* Linux 4.14+
* 支持 KVM

可以使用以下脚本检验：

```
err=""; \
[ "$(uname) $(uname -m)" = "Linux x86_64" ] \
  || err="ERROR: your system is not Linux x86_64."; \
[ -r /dev/kvm ] && [ -w /dev/kvm ] \
  || err="$err\nERROR: /dev/kvm is innaccessible."; \
(( $(uname -r | cut -d. -f1)*1000 + $(uname -r | cut -d. -f2) >= 4014 )) \
  || err="$err\nERROR: your kernel version ($(uname -r)) is too old."; \
dmesg | grep -i "hypervisor detected" \
  && echo "WARNING: you are running in a virtual machine. Firecracker is not well tested under nested virtualization."; \
[ -z "$err" ] && echo "Your system looks ready for Firecracker!" || echo -e "$err"
```

在正式环境中，官方推荐使用 jailer 运行 Firecracker。由于我们只是验证环境，故直接使用 Firecracker 二进制运行：

先下载最新版本：

```
latest=v0.13.0 && curl -LOJ https://github.com/firecracker-microvm/firecracker/releases/download/v${latest}/firecracker-v${latest}
```

**注：由于 v0.14.0 会出现 Firecracker API hang 问题，所以我这里还是选择 v0.13.0。**

接下来启动进程：

```
rm -f /tmp/firecracker.socket; ./firecracker --api-sock /tmp/firecracker.socket

```

开启另一个 shell，下载 kernel 和 rootfs：

```
curl -fsSL -o hello-vmlinux.bin https://s3.amazonaws.com/spec.ccfc.min/img/hello/kernel/hello-vmlinux.bin
curl -fsSL -o hello-rootfs.ext4 https://s3.amazonaws.com/spec.ccfc.min/img/hello/fsfiles/hello-rootfs.ext4
```

调用 Firecracker API，设置虚拟机 kernel 和 rootfs：


```
curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT 'http://localhost/boot-source'   \
    -H 'Accept: application/json'           \
    -H 'Content-Type: application/json'     \
    -d '{
        "kernel_image_path": "./hello-vmlinux.bin",
        "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"
    }'

curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT 'http://localhost/drives/rootfs' \
    -H 'Accept: application/json'           \
    -H 'Content-Type: application/json'     \
    -d '{
        "drive_id": "rootfs",
        "path_on_host": "./hello-rootfs.ext4",
        "is_root_device": true,
        "is_read_only": false
    }'
```

启动虚拟机：

```
curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT 'http://localhost/actions'       \
    -H  'Accept: application/json'          \
    -H  'Content-Type: application/json'    \
    -d '{
        "action_type": "InstanceStart"
     }'
```

这时回头看第一个 shell 窗口，已经连接了一个 TTY 设备到虚拟机。使用 root/root 登陆即可。

### 最后
笔者实际是想写 KataContainer 和 AWS Firecracker 的结合，以及它们之间会碰撞出怎么样的火花。不过总觉得需要先做些铺垫，简单介绍下基本原理和使用，故有了这篇充斥拿来主义的“水”文。先挖个坑，后面如果有时间会深入阐述 Firecracker 的实现原理。至于源码分析就算了，Rust 实在看的头疼。有经验的读者无需浪费时间，直接移步 github：https://github.com/firecracker-microvm/firecracker/blob/master/docs。


*《未完待续》*


