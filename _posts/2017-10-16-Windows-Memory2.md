---
layout: post
title: "A series used to talk about Windows- Memory Management 2"
date:  2017-10-16 22:05:00 +0800
---

这里接着之前的内容完成`Windows`内存管理的相关知识。

### Kernel-Mode Heaps(System Memory Pools)
在讲述完Windows所提供的一些基本功能之后，这里对一些重要的组成部分进程讲解。这里首先从内核的基本概念说起，来叙述内存管理器为内核提供的基本结构。

#### 内核内存管理区域
##### Pools
内存管理器为使用内核的对象提供了两个`pool`用来分配内存。
1. `Nonpaged pool`: 这个pool申请的内存将一直贮存在物理内存中，不会被交换出去，也就不会产生`page fault`。由于这个原因无论在哪个IRQL级别上的中断都可以访问该内存空间，且不会造成由于page fault需要在中断级别为`DPC\Dispatch`上处理，而造成系统崩溃。
2. `Paged pool`: 这部分内存可以被换出。由于driver大多时候不需要在`DPC\Dispatch`之上访问内存，所以可以使用这部分内存完成地址分配。

一个系统通常拥有多个`page pool`，从而减少分配和释放内存时需要更改全局变量而造成的系统开销。

##### Look-Aside List
Look aside list 由许多固定大小的block组成。通过look aside list可以快速的分配内存块。look aside list中包含一系列已经分配好大小的内存块。look aside list可以实现较快的内存分配速度，因为我们不需要使用自旋锁来完成内存分配任务。通常程序对于每一个处理器都会分配一个`look-aside list`，操作系统会根据分配好的数据块使用状况自动缩放`look-aside list`的大小。不同的进程可以分配自己的`look-aside list`。

### Heap Manager
上面所说的是对虚拟内存的分配情况。实际上我们在实际使用时并不会分配64KB的内存，在通常情况下我们只需要很小的一部分内存。为了优化对小内存分配的情形，Windows提供了专门的heap管理机制，用来管理应用程序对于小内存的使用。对于C/C++的函数来说，其`new/malloc/delete/free`即使用windows堆管理。最常用的堆管理函数有`HeapAlloc HeaFree HeapCreate HeapDestroy`等。

#### 堆的内部管理
每个应用程序都会有一个默认堆，默认堆的通常大小为1MB。线程可以通过调用`GetProcessHeap`来获得默认堆。Heap管理器分得的内存通常位于VirtualAlloc分得内存之中。
HeapManager在Application和Memory Manager之间有如下几层：
1. Windows Heap API
2. Front-End heap layer (optional)这个部分可以不需要
3. Core heap layer

Heap Manager运行多线程访问。但当访问线程过多时会有较大的开销。这个时候如果启用`Front-End layer`会使得分配性能得到较大的改善。这里支持的`Front-End layer`只有`Low Fragmentation Heap`一种。这种结构为了加快内存分配速度，把内存块分为预订大小的若干块，每种大小称作一个`bucket`。当分配内存时，会分配一个恰好能放下该大小的内存块给该线程。这些预定义的存储块最小是8 Byte，总共有128个大小最大是16384 Byte。这里结合`look-aside list`方法来确保每个bucket的数量。

同时对于较常访问的数据结构，操作系统将其分散到不同的`slot`中去。通常这些`slot`的大小为处理器的两倍，从而提高内存分配的效率。

#### Heap Security Features
为了防止对堆空间的修改，系统提出了一系列的对空间保护措施：
1. 堆的元数据被极高程度的随机化，从而防止对堆元数据的恶意修改；
2. 这些块在头部加入了一致性检查，从而防止简单的`corruptions`；
3. 最后，堆的基址`base address`也进行了简单的随机化。

#### Fault Tolerant Heap
这部分是个比较有意思的话题，Windows增加了Heap容错机制。这个机制使得Windows可以监控那些经常有堆使用错误的程序，当错误次数达到3词时就会加到观察列表。再有错误的时候会有一个client负责对堆进行迁移。迁移时候会有些增加堆大小，还有减缓内存释放等操作。

当然这个功能默认是不启用的，因为会产生一些性能上的开销