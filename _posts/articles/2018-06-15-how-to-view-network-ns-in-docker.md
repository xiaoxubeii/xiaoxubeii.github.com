---
layout: post
title: "How to View Network Namespace in Docker"
description: ""
category: articles
tags: [Docker, Kubernetes, Container, Network Namespace]
---
#  How to View Network Namespaces in Docker
When we start a container in Docker:

```
docker run -it nginx:1.9 sleep 1000
```
To get the container pid and view `/proc/$pid/ns`, we can find the namespace  created at container startup:

```
$ pid=$(docker inspect -f '{{.State.Pid}}' ${container_id})
$ ls /proc/$pid/ns -la
dr-x--x--x 2 root root 0 Jun 15 23:12 .
dr-xr-xr-x 9 root root 0 Jun 15 23:12 ..
lrwxrwxrwx 1 root root 0 Jun 15 23:13 ipc -> ipc:[4026539032]
lrwxrwxrwx 1 root root 0 Jun 15 23:13 mnt -> mnt:[4026539030]
lrwxrwxrwx 1 root root 0 Jun 15 23:12 net -> net:[4026539035]
lrwxrwxrwx 1 root root 0 Jun 15 23:13 pid -> pid:[4026539033]
lrwxrwxrwx 1 root root 0 Jun 15 23:13 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Jun 15 23:13 uts -> uts:[4026539031]
```

Unfortunately we cannot use `ip netns ls` to list network namespace. Why?

That is because `ip netns` will only search ns in `/var/run/netns/` directory. We can create symlinks manually:

```
$ ln -sfT /proc/$pid/ns/net /var/run/netns/$container_id
$ ip netns ls $container_id
8ac0b234be10
$ ip netns exec 8ac0b234be10 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
```
A alternative approach is to use `nsenter`:

```
$ nsenter -t $pid -n ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
```
