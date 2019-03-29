---
layout: post
title: "Introduction for KataContainers"
description: ""
category: articles
tags: [KataContainers, MicroVM]
---

# AWS Firecracker 和 KataContainers 初探（二）
## KataContainers
照例先贴官网简介：
> Kata Containers is an open source project and community working to build a standard implementation of lightweight Virtual Machines (VMs) that feel and perform like containers, but provide the workload isolation and security advantages of VMs.

简单总结： KataContainers 是使用轻量级虚拟机实现容器技术的开源项目，最早来源于 Intel® Clear Containers 和 Hyper runV。不过随着 Hyper 两位核心工程师加入蚂蚁金服，未来阿里在 KataContainers 会有很多的话语权。

![](/images/15537543761371/15537547378830.jpg)

KataContainers 的主要组件包括：runtime、proxy、shim、agent 和 hypervisor。

### Runtime
`kata-runtime`是一个 OCI container runtime 实现，它和`runc`是处于同一个层面，主要负责容器的生命周期管理，包括创建、启动、执行和删除等。不过`runc`是使用 os 级别的技术实现容器，比如 Linux 的 namespace、cgroup 和 chroot 等，而`kata-runtime`是使用 VMM 创建虚拟机实现容器。

![](/images/15537543761371/15538400523928.jpg)

### Proxy
`kata-proxy`是外部和虚拟机或虚拟机内部容器间通信的代理，比如使用 `virtio-serial`或者`vsock`。不过在 KataContainers 1.5 以后，它的功能应该全部整合进了`containerd-shim-kata-v2`这个组件。

### Shim
熟悉 Kubernetes 或 containerd 的读者应该对`shim`不会陌生，`shim` 主要是作为容器的适配和代理。这块历来都是比较乱的，因为各种组织和厂家都在主推自己的标准，这导致出现了各式各样的 shim。比如 `containerd-shim`、`CRI-O conmon` 和 `CRI shim`等。并且由于传统的 `containerd-shim` 只能管理进程级别的容器，而无法管理虚拟机级别的容器。所以 KataContainers 又实现了自有的 `kata-shim`。不过 KataContainers 也实现了 [Containerd Runtime V2 (Shim API)](https://github.com/kata-containers/runtime/tree/master/containerd-shim-v2) 接口，那么在这种情况下，只需要一个 `containerd-shim-kata-v2` 组件即可。

![](/images/15537543761371/15538423093839.jpg)

### Agent
`kata-agent`是在虚拟机内运行的代理，用于管理容器和容器内进程。`kata-agent`可在一个虚拟机中运行一个或多个容器。比如在与 Kubernetes 集成中，一个 pod 对应一个虚拟机 sandbox，并且在其中可以运行多个容器。与 docker 集成中，`kata-agent`则只创建一个容器。

在目前版本中，`kata-agent`在虚拟机内部是调用`libcontainer`管理容器，这也是`runc`的实现方式。

### Hypervisor
前面讲过，`KataContainers`的核心是使用 VMM 创建虚拟机，并在虚拟机内部创建容器。`KataContainers`设计之初是支持多 VMM，也就是多  hypervisor 的，这块是通过 [virtcontainers project](https://github.com/containers/virtcontainers) 实现。目前版本是支持 QEMU/KVM 和 AWS FireCracker。

### 总结
目前在市面上，实现虚拟化容器的主要项目有 runv、ClearContainers、gVisor、KataContainers 和 AWS FireCracker。runv 和 ClearContainers 整合进了 KataContainers。Google 的 gVisor 严格来讲不是传统意义上的 VMM。而 AWS FireCracker 侧重于对 VMM 和虚拟化的实现，在容器生态的支持上还处于非常初级的阶段。所以 KataContainers 无论在方案的成熟度和生态的支持上，都占有较大的优势。不过抛开虚拟化和生态，KataContainer 实际并无太多核心技术。它做的主要工作是使用虚拟机实现容器接口，这在实现上并无太大难度。KataContainers 未来若想发展就必须把住生态入口，制定标准，并和几个大的开源项目和组织，比如 Kubernetes、Containerd 和 CNCF 等紧密合作。不过在国内，随着蚂蚁金服和阿里的介入，KataContainers 的主导权会逐渐向这些公司倾斜，这使得其他公司（主要是国内的竞争对手）在技术选型上会有所顾忌。笔者不是特别看好 KataContainers 的发展，当然未来还是拭目以待。
