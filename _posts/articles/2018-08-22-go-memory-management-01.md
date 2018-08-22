---
layout: post
title: "Golang Memory Management Part I: Introduce Memory"
description: ""
category: articles
tags: [Linux, GO, Memory Management]
---

# Golang 内存模型原理浅析：（一）内存简介
## 何为内存？
先看维基百科的定义：
> 随机存取存储器（英语：Random Access Memory，缩写：RAM），也叫主存，是与CPU直接交换数据的内部存储器。[1]它可以随时读写（刷新时除外，见下文），而且速度很快，通常作为操作系统或其他正在运行中的程序的临时数据存储媒介。

内存实质上就是一串存储单元，以 8 bit 为单位存储信息。CPU 要想访问这些信息，就必须知道它们的位置，这些类似门牌号的东西就是内存地址。
![](/images/15348326726886/15348344928477.jpg)    

如今操作系统多为多任务，可想而知，如果每个任务（也就是进程）都可以直接访问物理内存，那么任务的数据就没有隔离性和安全可言。所以为了隔离内存空间，又引入了虚拟内存的概念。

## 虚拟内存和地址转换
一个程序在操作系统中运行时，它会拥有自己的虚拟内存，并认为自己是独享整个内存空间。程序代码通过虚拟内存空间访问物理内存，从虚拟内存地址到物理内存地址的转换被称为地址映射。
![](/images/15348326726886/15348351968396.jpg)

这个地址映射的过程因内存管理方式的不同而有所差异，一种常用的方式是段页式管理。

在 Intel 平台下，逻辑地址（logical address）是 selector:offset 这种形式，selector 是 CS 寄存器的值，offset 是 EIP 寄存器的值。如果用 selector 去全局描述符表（GDT）里拿到段基址（segment base address），然后加上段内偏移（offset），这就得到了线性地址（linear address）。我们把这个过程称作段式内存管理。

如果再把线性地址切成四段，用前三段分别作为索引去 PGD、PMD、Page Table 里查表，最终就会得到一个页表项(Page Table Entry)，那里面的值就是一页物理内存的起始地址，把它加上线性地址切分之后第四段的内容(又叫页内偏移)就得到了最终的物理地址。我们把这个过程称作页式内存管理。在计算机中有专门的芯片 Memory Management Unit（MMU），可用于页表转换，它位于 CPU 和内存之间。
![](/images/15348326726886/15348451415807.jpg)

大家可能会感到奇怪，为什么上面没有提到虚拟地址空间（virtual address）。其实在 Intel IA-32 手册里并没有提到这个术语，但是在内核的确是用到了这个概念，比如 __va 和 __pa 这两个宏定义。其实虚拟地址是程序里使用的地址，也就是一个指针值，指针的本质就是 EIP 寄存器中的值。而前面提过逻辑地址中的偏移量就是 EIP 寄存器的值，也就是说虚拟地址 = 逻辑地址偏移量。而在 Linux 中，内核会将段基址设为0，也就相当于不分段了，所以在值上虚拟地址 = 线性地址。
## 程序是什么？
当你在操作系统中编译程序，比如 GO，它会被编译成一个可执行机器代码文件或者库文件。这些文件是按照一定的格式生成的，比如 Linux 的 Executable and Linkable Format（ELF）或 Windows 的 Portable Executable。可执行文件除了包含一些应用的元数据，比如操作系统架构、debug 信息外，还包含以下部分：.text（代码段）、.data（全局变量）和.rodata（全局常量）。

操作系统中有个模块叫程序加载器，可用于执行程序。在 Linux 中，我们可以通过系统调用`execve()`加载程序。当加载过程运行时，会执行以下步骤：

1. 验证程序镜像
2. 程序镜像从磁盘加载到内存
3. 将命令行参数传给调用栈
4. 初始化寄存器
5. 控制权交给程序代码

当静态的程序运行后，就变成了一个进程。进程在内存映象中会分成以下几个部分：
![](/images/15348326726886/15348655066835.jpg)

* text：程序代码
* data：已初始化的全局变量和数据
* bss：未初始化的全局变量和数据
* stack：栈
* heap：堆

在这里我们需要注意的是，此处的段和前面提到的 Intel IA-32 物理段并没有关系。这里的每个段都对应一个 VMA（Virtual Memory Areas），由内核负责分配。

## 一个示例
我编写了一个 Golang 示例，用于说明 Linux 中程序内存段的分配：

```
package main

import (
    "fmt"
    "os"
)

var a int
var b = 123

func main() {
    e := d()
    fmt.Printf("pid is %v.\n", os.Getpid())
    fmt.Printf("text segment is %v.\ndata segment is %v.\nbss segment is %v.\nheap segment is %v.\n", d, &b, &a, &e)

    var a [1]int
    print("stack segment is ", &a)

    for {
    }
}

func d() (*int) {
    c := 0
    return &c
}
```

执行以下命令：

```
$ go run -gcflags '-m -l' main.go

# command-line-arguments
./main.go:25:12: &c escapes to heap
./main.go:24:5: moved to heap: c
./main.go:13:41: os.Getpid() escapes to heap
./main.go:14:16: d escapes to heap
./main.go:14:106: &b escapes to heap
./main.go:14:106: &b escapes to heap
./main.go:14:110: &a escapes to heap
./main.go:14:110: &a escapes to heap
./main.go:14:114: &e escapes to heap
./main.go:14:114: &e escapes to heap
./main.go:12:5: moved to heap: e
./main.go:13:15: main ... argument does not escape
./main.go:14:15: main ... argument does not escape
./main.go:17:32: main &a does not escape
pid is 15883.
text segment is 0x483c10.
data segment is 0x5150c0.
bss segment is 0x544e40.
heap segment is 0xc42009a018.
stack segment is 0xc420057f10

```

在 Linux 下，我们可以通过`/proc/{pid}/maps`获取该进程的内存分段：

```
$ cat /proc/15883/maps | cut -f1 -d' '
00400000-00484000 // text segment
00484000-00515000
00515000-00529000 // data segment
00529000-00548000 // bss segement
c000000000-c000001000
c41fff8000-c420100000 // heap or stack segment?
7ff16fd7d000-7ff16fe1d000
7ffd96b69000-7ffd96b8a000
7ffd96bd1000-7ffd96bd3000
ffffffffff600000-ffffffffff601000
```

不过在这里我发现一个问题，栈和堆的地址落在了一个 VMA 里。按照理论它们应该归属于不同 VMA。目前我还未找到问题原因，后续会继续关注。



