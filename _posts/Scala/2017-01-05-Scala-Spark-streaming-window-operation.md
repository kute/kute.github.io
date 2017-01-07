---
published: true
layout: post
title: 理解spark streaming window operation
category: Scala
tags: Spark
time: 2017.01.06 14:22:00
keywords: 
description: 理解 spark streaming window operation 

---

## 基础概念
  在`window-operation`中有三个概念：
  
  - streaming interval 
  - window length
  - slide inteval
  
## 总结

> 以slide interval间隔，来计算window length内的数据（如下面的例子就是，每隔10s窗口滑动一次，然后计算30s内的数据）

## 示例

> 下面是基于 streaming interval=1s，windows length = 30s，slide interval=10s（后两者得是第一个的整数倍）

下面控制台的输出，一个30s大小的窗口

![图片链接](/public/img/posts/scala/spark-window-console.jpg)

下面是流程图

![图片链接](/public/img/posts/scala/spark-window-flow.png)

过程描述（可以对比控制台输出）：

    1. 当程序启动，在第一个10s到来时，也就是t1时间段内，我输入了数据“kute is bai”，窗口目前滑动到t1，如红色框（不考虑之前的过程，暂且认为是刚开始滑动）只包含了t1时间段，所以计算t1时间段内的数据，也就是“kute is bai”，所以在第一次控制台输出三个单词都是1。
    2. 随着时间流失，第二个10s马上来到，但是在t2时间段内，无数据进来，所以当窗口滑动到t2，如绿色框只包含了t1和t2两个时间段，因为t2无数据，t1是之前进来的数据，所以计算此时窗口内的数据（t1+t2），三个单词的数量还是1。
    3. 紧接着第三个10s来到，此时我在t3时间段内输入数据“kute is bai”（在控制台中可以看到Block Input，就是在输入数据，至于我到底在哪几秒输的数据无所谓，总之是在20~30s之间输入的），此时窗口滑动到t3，如蓝色框，现在蓝色框包含了t1，t2，t3三个时间段，所以计算窗口内的数据（也就是t1 + t2 + t3），所以三个单词的数量都是2。
    4. 现在第四个10s来到了，同理，窗口滑动到了t4，如黄色框，但是此时只计算t2 + t3 + t4的数据，因为窗口的大小是30s（也就是3个time unit）所以只能计算30s之内的数据，所以就只有t2~t4了，但是因为t4时间段内我又输入了数据“kute is bai”，所以此时t2 + t3 + t4的数据为：三个单词的数量还是2。
    5. 第五个时间段来了，窗口滑动到t5（注意只有当时间到达了slide interval末尾控制台才会计算的），如紫色框，窗口大小不变还是3个time unit，包含了t3, t4, t5，所以三个单词的数据为2。
    6. 同理，到了t6，只包含了t4, t5, t6，所以数据为1。

窗口的滑动意味着时间的流逝，所以这个窗口的操作就是，每隔10s，窗口滑动一次，去计算过去30s的数据（过去的数据在内存）