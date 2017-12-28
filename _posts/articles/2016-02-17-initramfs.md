---
layout: post
title: "创建自定义initramfs"
description: ""
category: articles
tags: [Linux initramfs]
---
在Linux的启动文件中，除了名为vmlinuz的kernel内核外，还有一个名为initramfs的文件。initramfs是基于内存filesystem，实际它就是一个通过gz压缩的cpio归档文件：

    # file initramfs    
    initramfs.img: gzip compressed data, was "initramfs", from Unix, last modified: Sat Jan 23 01:29:35 2016, max compression

## 提取initramfs

想要自定义initramfs，首先需要解压缩：

    # mv initramfs.img initramfs.gz && gunzip initramfs.gz

然后使用cpio命令将文件提取出来：

    # mkdir output && mv initramfs output && cd output 
    # cpio -idv < initramfs && rm -f initramfs

然后进入相应文件夹修改文件即可。

## 制作initramfs
修改完文件，想要重新制作initramfs.img，我们需要先归档：

    find . -print | cpio -H newc -o > ../initramfs

归档完成后，我们还需要将归档文件压缩成gz：

    gzip -9 initramfs
