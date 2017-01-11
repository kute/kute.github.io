---
published: true
layout: post
title: 多线程,多进程,协程 处理任务模块-Eventor
category: Python
tags: Python Coroutine Thread Process
time: 2017.01.05 14:22:00
keywords: 
description: 方便加快处理IO密集型的任务

---

## 介绍

  Eventor是基于`gevent`, `multiprocessing`编写, 为加快处理`IO密集型任务`的python模块, 目前只有`协程`与`多进程`, 至于`多线程`还不如用`协程`呢, `多进程`是为了利用`多核`, 如此就可以
使用`多进程+协程`的方式,更快的处理任务, 未来还会把其他对于线程,异步IO的比较好的模块如`asyncio`等加进来, 主要也是在工作中经常有这样的批量任务所以写个"脚本"来方便执行.

## 例子

  目前模拟了三个场景:
  
#### 场景-

  比如我又一堆数据的集合, 每个数据都要经过某个转换(这样的场景可以衍生为 `对某个接口URL或者一堆URL进行调用`)
  
    >>> from eventor import Eventor
    >>> elelist = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    >>> func = lambda x: x + 10
    >>> e = Eventor(threadcount=3, taskunitcount=3, func=func, interval=1)
    >>> result = e.run_with_tasklist(elelist)
    >>> print(result)
    [11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
    
  上述例子是开启3个线程,将任务(共有10个task)分割为每份3个task执行的, 间隔1s
  
#### 场景二

  和场景一类似, 只不过数据是已文件的形式存在
  
    >>> from eventor import Eventor
    >>> file = "test/data.txt"
    >>> func = lambda x: int(x) + 10
    >>> e = Eventor(threadcount=3, taskunitcount=3, func=func, interval=1)
    >>> result = e.run_with_file(file)
    >>> print(result)
    [11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
    
#### 场景三

  多个消费者消费共享数据, 数据不重复消费
  
    >>> from multiprocessing import cpu_count
    >>> from eventor import start_multi_consumer
    >>> consumer_count = 4
    >>> def confunc(data):
    ....    print("process[{}] deal with {}".format(multiprocessing.current_process().name, data))
    >>> datalist = [1, 2, 5, 3, 6, 8, 23, 'data', 232]
    >>> start_multi_consumer(consumercount=consumer_count, iterable=datalist, consumer_func=confunc)
    process[ForkPoolWorker-3] deal with 1
    process[ForkPoolWorker-2] deal with 2
    process[ForkPoolWorker-4] deal with 5
    process[ForkPoolWorker-5] deal with 3
    process[ForkPoolWorker-3] deal with 6
    process[ForkPoolWorker-2] deal with 8
    process[ForkPoolWorker-4] deal with 23
    process[ForkPoolWorker-5] deal with data
    process[ForkPoolWorker-3] deal with 232
    
  上述例子是, 开启 4 个进程, 每个进程打印 当前进程名以及准备消费的 共享数据 (在处理稍微比较繁琐的任务时, 可以和上面结合, 也就是 `多进程 + 协程`, 这样既能利用多核, 又能利用协程高效执行)
  
项目地址以及API: [https://github.com/kute/eventor](https://github.com/kute/eventor)

持续更新....