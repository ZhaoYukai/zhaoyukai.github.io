---
layout: post
title: Fresco如何“偷”Android内存？
date: 2016-01-13
categories: blog
tags: [Android性能优化][内存OOM]
description: Tricking Android MemoryFile

---

之前在做一个内存优化的时候，使用到了MemoryFile，由此发现了MemoryFile的一些特性以及一个非常trickly的使用方法，因此在这里记录一下。

<h1>What is it</h1>

MemoryFile是android在最开始就引入的一套框架，其内部实际上是封装了android特有的内存共享机制Ashmem匿名共享内存。

简单来说，Ashmem在Android内核中是被注册成一个特殊的字符设备，Ashmem驱动通过在内核的一个自定义slab缓冲区中初始化一段内存区域，然后通过mmap把申请的内存映射到用户的进程空间中（通过tmpfs），这样子就可以在用户进程中使用这里申请的内存了。

另外，Ashmem的一个特性就是可以在系统内存不足的时候，回收掉被标记为”unpin”的内存，这个后面会讲到。

另MemoryFile也可以通过Binder跨进程调用来让两个进程共享一段内存区域。由于整个申请内存的过程并不再Java层上，可以很明显的看出使用MemoryFile申请的内存实际上是并不会占用Java堆内存的。
MemoryFile暴露出来的用户接口可以说跟他的名字一样，基本上跟我们平时的文件的读写基本一致，也可以使用InputStream和OutputStream来对其进行读写等操作：

    MemoryFile memoryFile = new MemoryFile(null, inputStream.available());
    memoryFile.allowPurging(false);
    OutputStream outputStream = memoryFile.getOutputStream();
    outputStream.write(1024);

上面可以看到allowPurging这个调用，这个就是之前说的”pin”和”unpin”，在设置了allowPurging为false之后，这个MemoryFile对应的Ashmem就会被标记成”pin”，那么即使在android系统内存不足的时候，也不会对这段内存进行回收。另外，由于Ashmem默认都是”unpin”的，因此申请的内存在某个时间点内都可能会被回收掉，这个时候是不可以再读写了。

<h1>Tricks</h1>
MemoryFile是一个非常trickly的东西，由于并不占用Java堆内存，我们可以将一些对象用MemoryFile来保存起来避免GC，另外，这里可能android上有个BUG：

在4.4及其以上的系统中，如果在应用中使用了MemoryFile，那么在dumpsys meminfo的时候，可以看到多了一项Ashmem的值：

![github](http://mmbiz.qpic.cn/mmbiz/e4JibCgzXv6Q8Q5loNoKgDtqXMcEn2DJ7X4tGCj0ux6SGevzI3V9HT9LMmscROWaw0RDWsgHBl7zYS1tdYAkw7w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1 "github")




