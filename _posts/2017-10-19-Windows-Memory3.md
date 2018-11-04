---
layout: post
title: "A series used to talk about Windows- Memory Management 3 (draft)"
date:  2017-10-19 21:08:00 +0800
---

这里是`Windows`内存管理的第三部分。

### Virtual Address Space Layouts
这部分大家都比较熟悉，是虚拟地址空间到物理地址空间的映射。

当然，虚拟地址空间分为用户空间和内核空间。内核内存空间通常会包含这几个部分：
1. System code
2. Nonpaged pool
3. Paged pool
4. System cache
5. System page table entries(PTEs) 这里只映射system pages
6. System working set lists。这里有三个system working set (cache working set, paged pool working set 以及system PTEs working set)
7. System mapped view。用来map Win32k.sys的
8. Hyperspace
9. Crash dump information
10. HAL usage

对于系统地址而言，大部分类型的地址空间是动态分配的，并没有明确指定每块空间的分配地址。
其中Session Space就是这样一种地址空间。这个Session地址空间是系统空间的一部分，所有处于这个session的进程都共享这个地址空间。下面是session地址空间的分配图：

![Session mempry Pic](/media/image/Session_layout.PNG "Session 地址空间分配")

64bit 的内存映射方式和32位有着很大的不同。与32位将整个内存分为两部分不同，64为系统将内存分为好几个部分。下面的图中列出了64bit的内存空间分配方式。
![Session mempry Pic](/media/image/64bit_address_space_size.PNG "64bit系统地址空间大小")

对于64bit的地址空间来说，最大可以访问16EB的地址。目前的处理器只实现了低48 bit，也就是可以访问256TB的地址空间。而目前Windows只允许使用16TB的地址空间。其中用户进程使用8TB而系统进程使用另外的8TB
Windows操作系统有一些使用64bit的限制，由于之前更具处理器位数进行优化，有些原子操作需要相应的位数才能支持。64bit后由于结构的限制无法支持48 bit，故目前只支持到16TB，同时保持操作系统运行的高性能。