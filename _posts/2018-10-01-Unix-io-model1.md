---
layout: post
title: "Unix IO 模型"
date:  2018-10-01 19:23:00 +0800
---
# Unix IO Model - Part I

Unix 网络编程的作者把io分为阻塞io，非阻塞io，io复用，信号驱动式io以及异步io。由于最近在把一个项目从同步的方式转成异步的形式（主要是为了处理io调用的性能问题），所以要了解和复习一下这些io的实现和原理。


## 阻塞与非阻塞，同步与异步
这个问题是在之前一直困惑的问题。这里采用`unix 网络编程的描述`说下阻塞，非阻塞，同步，异步的区别。

首先阻塞、非阻塞的概念和同步异步的概念没有任何关系。阻塞非阻塞是对一个线程来说的，而同步异步是对函数调用来说的。

### 阻塞io
阻塞式io是我们在平时使用的最多的一种io操作。这种io表型为调用线程会一直阻塞，直到io操作返回。平时使用的io操作比如：`read`，`write`都是阻塞式io。

这种io形式操作简单，但由于会造成线程无谓等待，尤其在存在大量io操作的情况下会导致性能问题。

### 非阻塞io
与阻塞式io对应，线程不会阻塞直至io操作返回。非阻塞io的解决方案有多种，一种可以通过`fcntl(fd, F_SETFL, flags | O_NONBLOCK);`将文件描述符设置为非阻塞的形式。这时如果io操作没有完成，调用相应的函数会返回`EWOULDBLOCLK`错误，线程继续执行。
另一种形式是io复用。io复用将所有io操作都阻塞到一个文件描述符上，只有当有对应的读写操作发生时才会返回，这样真正的处理进程并不会阻塞到io操作上。当然异步io也是非阻塞的形式。

### 同步io调用
同步调用是指一种io函数调用。这里特指函数调用是同步的，即函数调用会返回调用结果。`read`，`write`，`select`，`poll`都是一种同步的形式。

### 异步io调用
wiki百科是这么定义异步io：`asynchronous I/O (also non-sequential I/O) is a form of input/output processing that permits other processing to continue before the transmission has finished.`[1]。往往异步调用并不是我们所要的真正返回。我们在使用异步调用的时候常常会设置一个`callback`，当异步调用处理完后触发这个`callback`来完成后续操作。我们不知道这个操作何时触发，这和同步调用我们主动调用相应的io处理操作有较大的区别。

目前常见的unix异步调用有`aio_read`，`aio_write`。

## 实现原理简介
io调用一般都为系统调用，有关系统调用的讨论在[System call internal](https://binyang2014.github.io/2018/10/02/System-call-internal.html)中已经进行了简单的叙述。在这里我们主要讨论一下Unix中io复用的不同机制以及异步io的实现。

### IO复用
#### 概念
IO复用是一种使用一个文件描述符监听多个文件描述符状态的方式。io复用的最大优势是程序不必阻塞在多个io调用中，只阻塞在`select`，`epoll`等系统函数。当所监听的文件描述符已经就绪则会返回，从而使程序进行后续的操作。
#### 几种io复用类型
常用的io复用方式有`select`,`poll`等。不同的操作系统又推出了自己的高性能io复用方式。比如`epoll`(Linux)，`kqueue`(BSD)等。
#### select和epoll
这里以`Linux`为例，说一下`select`,`epoll`等系统函数的实现机制。

这里我们想要解答几个问题：

1. `select`和`epoll`是如何监听文件描述符的状态
2. `select`和`epoll`为什么会有性能上的差异

首先我们看下`select`系统调用的实现方式

### 异步io

#### aio_read & aio_write
#### Windows IOCP

## References

[1] https://en.wikipedia.org/wiki/Asynchronous_I/O