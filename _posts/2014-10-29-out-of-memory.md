---
layout: post
title: Java内存溢出错误：OutOfMemoryError异常分析
categories: Java
description: OutOfMemoryError异常分析
keywords: OutOfMemoryError
---

在JVM的运行时数据区，除了程序计数器之外，其他区域都有可能会产生OutOfMemoryError异常。

# 1. Java堆溢出

Java堆溢出时会报下面的异常错误：

```
java.lang.OutOfMemoryError: Java heap space
```

在启动虚拟机的时候可以加上参数：`-XX:+HeapDumpOnOutOfMemoryError`，让虚拟机在出现内存溢出异常时Dump出当前的内存堆转储快照。出现堆溢出后，可以通过内存映像分析工具（如Eclipse Memory Analyzer）对堆转存快照进行分析：

首先要分析清楚到底是内存泄露（Memory Leak）还是内存溢出（Memory Overflow）。

- 如果是内存泄露，说明对象无法被GC回收。通过工具可以查看对象的引用链，检查在引用链的路径中是否有不正确的使用导致无法被GC回收。
- 如果不是内存泄露，对象确实还必须存活，就需要检查虚拟机的参数（-Xmx,-Xms）和物理机内存是否可以调大；代码中检查部分对象是否生命周期过长等情况。

# 2. 虚拟机栈和本地方法栈溢出

在Hotspot虚拟机中不区分虚拟机栈和本地方法栈，栈容量都由参数-Xss来设定。在栈溢出的规定中有两种异常：

1. 如果线程请求的深度大于虚拟机所允许的最大深度，将抛出StackOverflowError
2. 如果虚拟机在扩展栈时无法申请到足够的空间，将抛出OutOfMemoryError

不过在实际实验中，在单线程下，无论是栈帧太大，还是虚拟机栈容量太小，当内存无法分配的时候，都是抛出StackOverflowError。而在多线程下，线程过多耗尽内存则抛出OutOfMemoryError异常。

假设操作系统有剩余内存2G，减去Xmx（最大堆容量），再减去MaxPermSize（最大方法区容量），程序计数器消耗很少可忽略，不算虚拟机本身消耗的内存，剩下的就是栈（虚拟机栈+本地方法栈）的内存了。剩下的内存就这么大，每个线程的栈容量越大，可建立的线程就越少。这时把内存耗尽就会产生OutOfMemoryError异常。

```
java.lang.OutOfMemoryError: unable to create new native thread
```

由于多线程导致的内存溢出，在不能减少线程数或者增加系统内存的情况下，只能通过减少最大堆和减少栈容量来换取更多线程。

上述异常发生可以有以下几个方向解决：

- 最根本的问题是，是否真的需要这么多的线程，能否使用线程池来控制线程数量
- 增加系统内存（进程限制的最大内存），如使用64位系统、修改线程限制
- 减小栈容量，JVM默认1024k的栈容量，一般不需要这么大，使用-Xss来修改
- 减小堆大小，使栈能从进程中得到的内存增大，用-Xmx参数来修改
- 减小方法区大小，原理同上，用-XX:MaxPermSize参数修改

# 3. 运行时常量池溢出

运行时常量池属于方法区（Hotspot虚拟机的持久代），可以通过-XX:MaxPermSize、-XX:PermSize参数来设置持久代大小。所以常量池溢出就是抛出OutOfMemoryError异常并说明是在持久代内存溢出了。

```
java.lang.OutOfMemoryError: PermGen space
```

# 4. 方法区溢出

方法区存放Class的相关信息，通过常量池的情况可以知道，方法区在Hotspot虚拟机的持久代，溢出时抛出的溢出和常量池是一样的

```
java.lang.OutOfMemoryError: PermGen space
```

# 5. 本机直接内存溢出

直接内存可以通过`-XX:MaxDirectMemorySize`指定，默认是与`-Xmx`一样大。DirectByteBuffer类可以从本机内存来分配，抛出内存溢出只显示OutOfMemoryError异常：

```
java.lang.OutOfMemoryError
```
