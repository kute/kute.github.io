---
published: true
layout: post
title: Zookeeper简介
category: Java
tags: Java Service
time: 2017.01.05 14:22:00
keywords: 
description: Zookeeper简介

---

### 介绍


> 开源分布式应用协调服务
> 它通过提供基于类似于文件系统的目录节点树方式的数据存储，来维护和监控你存储的数据的状态变化，通过监控这些数据状态的变化，从而可以达到基于数据的集群管理。

总结：

- 类文件系统的数据模型
- 基于Watcher机制的分布式事件通知
- 高容错数据一致性协议
- 数据模型与层级结构

![图片链接](/public/img/posts/java/zk_struct.png)
 
1. 分层的文件系统目录树结构（不同于文件系统的是，节点可以有自己的数据，而文件系统中的目录节点只有子节点）

2. 每个节点称之为 znode，且可以有多个版本，意味着同一节点可以存储多份数据

 

### 基本概念

 

1、持久节点

该数据节点被创建后，就会一直存在于zookeeper服务器上，直到有删除操作来主动删除这个节点。

2、临时节点

临时节点的生命周期和客户端会话绑定在一起，客户端会话失效，则这个节点就会被自动清除。

3、持久顺序节点

在ZK中，每个父节点会为他的第一级子节点维护一份时序，会记录每个子节点创建的先后顺序。基于这个特性，在创建子节点的时候，可以设置这个属性，那么在创建节点过程中，ZK会自动为给定节点名加上一个数字后缀，作为新的节点名。这个数字后缀的范围是整型的最大值

4、临时顺序节点 

5、ZXID

事务ID，对于每个事务请求，zk都会为其分配一个全局唯一的事务ID，即ZXID，这里的事务一般包括数据节点的创建与删除、数据节点内容更新和客户端会话创建与失效等操作

总体结构

Zookeeper集群(2n+1个服务允许n个失效，也就是  【n/2 + 1】个仍可提供服务)。Zookeeper服务主要有两个角色，一个是leader，负责写服务和数据同步，剩下的是follower，提供读服务，leader失效后会在follower中重新选举新的leader。

![图片链接](/public/img/posts/java/zk_total_struct.png)


1. 客户端可以连接到每个server，每个server的数据完全相同。

2. 每个follower都和leader有连接，接受leader的数据更新操作。

3. Server记录事务日志和快照到持久存储。

4. 大多数（n/2 + 1）server可用，整体服务就可用。

5. 三种角色


角色 | 职责
------------ | -------------
leader |负责所有事务写操作，会话状态以及节点数据变更同步，保证数据顺序处理
follower | 处理非事务请求；参与leader选主投票
observer|不参与投票；只提供读服务

### 特点

- 顺序一致性按照客户端发送请求的顺序更新数据。

- 原子性：更新要么成功，要么失败，不会出现部分更新。

- 单一性 ：无论客户端连接哪个server，都会看到同一个视图。

- 可靠性：一旦数据更新成功，将一直保持，直到新的更新。

- 及时性：客户端会在一个确定的时间内得到最新的数据。

### 权限控制

zookeeper使用ACL来控制对znode节点的访问

1、格式：scheme:id:permissions

名称 | 说明
---|---
scheme:id | world: 它下面只有一个id, 叫anyone, world:anyone代表任何人，zookeeper中对所有人有权限的结点就是属于world:anyone的。auth: 它不需要id, 只要是通过authentication的user都有权限（zookeeper支持通过kerberos来进行authencation, 也支持username/password形式的authentication)。digest: 它对应的id为username:BASE64(SHA1(password))，它需要先通过username:password形式的authentication。ip: 它对应的id为客户机的IP地址，设置的时候可以设置一个ip段，比如ip:192.168.1.0/16, 表示匹配前16个bit的IP段。super: 在这种scheme情况下，对应的id拥有超级权限，可以做任何事情(cdrwa)。
permissions | CREATE(c): 创建权限，可以在在当前node下创建child node。DELETE(d): 删除权限，可以删除当前的node。READ(r): 读权限，可以获取当前node的数据，可以list当前node所有的child nodes。WRITE(w): 写权限，可以向当前node写数据。ADMIN(a): 管理权限，可以设置当前node的permission。
setAcl /zookeeper/node1 world:anyone:cdrw

### 工作模式（3种）

1、Standalone：单点模式，有单点故障问题

2、伪分布式：在一台机器同时运行多个ZooKeeper实例，仍然有单点故障问题，当然，其中配置的端口号要错开的，适合实验环境模拟集群使用

3、完全分布式：在多台机器上部署ZooKeeper集群，适合线上环境使用

### 典型场景

> 总的来说：zk会把数据全量存储在内存中以提高读取性能，故而最适于读多写少且轻量级数据（默认设置下单个dataNode限制为1MB大小）的应用场景

#### 1、数据发布与订阅

          客户端向服务端注册自己需要关注的节点，一旦该节点的数据发送变更，那么服务端就会向相应的客户端发送Watcher事件通知，客户端接收到这个消息通知之后，需主动到服务端获取最新的数据

![图片链接](/public/img/posts/java/zk_store.png)    

     注意点：

              a. 数据量通常比较小。

              b. 数据内容在运行时会发生动态变化。

              c. 集群中各机器共享，配置一致


#### 2. 命名空间服务

           分布式命名服务，创建一个节点后，节点的路径就是全局唯一的，可以作为全局名称使用。（对比JNI）

           如：

                 分布式服务框架Dubbo中使用ZooKeeper来作为其命名服务，维护全局的服务地址列表

![图片链接](/public/img/posts/java/zk_dubbo.png)

#### 3. 分布式通知/协调

          不同的系统都监听同一个节点，一旦有了更新，另一个系统能够收到通知。

           如：心跳检测

基于ZooKeeper的临时节点特性，可以让不同的机器都在ZooKeeper的一个指定节点下创建临时子节点，不同的机器之间可以根据这个临时节点来判断对应的客户端机器是否存活。通过这种方式，检测系统和被检测系统之间并不需要直接相关联，而是通过ZooKeeper上的某个节点进行关联，大大减少了系统耦合

#### 4. 分布式锁

          使用ZK构建分布式锁都是利用了ZK能够保证高并发情况下，子节点创建的唯一性

a. 独占锁（排他锁）：

![图片链接](/public/img/posts/java/zk_lock.png)

扩展：

a. 分布式锁服务关键技术和常见解决方案

b. google chubby

#### 5. 集群管理

每个加入集群的机器都创建一个节点，写入自己的状态。监控父节点的用户会受到通知，进行相应的处理。离开时删除节点，监控父节点的用户同样会收到通知。

### 监控

1、监控方式有两种

a. 4字命令 

echo ruok | nc 127.0.0.1 5111

nc unix command

b. JMX

c. JConsole

d. TaoKeeper

 

### 原理介绍

#### 强一致性

0、分布式CAP理论

数据存在的节点副本越多，容忍性越高；副本越多，需要拷贝更新的数据越多，一致性就难以保证；为了保证一致性，需要等待全部节点更新完成，这时可用性就会降低

P是必须的，取舍A和C

1、什么是 一致性

在分布式环境中，数据的多个副本保持一致

2、ZAB协议（Zookeeper原子消息广播协议）

工作模式

内容

崩溃恢复

选主（PK ID 和ZXID）+数据同步（写事务日志，提出和提交事务ACK）：

任何时候都需要保证只有一个主进程负责进行事务操作，而如果主进程崩溃了，就需要迅速选举出一个新的主进程。主进程的选举机制与事务操作机制是紧密相关的

消息广播

leader将请求生成事务，广播给所有follower，待收到大多数ACK时，再广播 commit消息，follower收到后再应用事务操作

 

### 2N+1

zookeeper特性：集群中只要有过半的机器是正常工作的，那么整个集群对外就是可用的

1、为什么是 2N + 1 （奇数）?

2N个server 允许 N-1 个 挂掉，即容错性 为 N -1.

2N + 1 个 server允许 N 个挂掉, 即容错性为 N -1.

换句话说：2N和 2N - 1的 容错性都是 N -1.

过半原则：Zookeeper的大多数行为都是基于投票的，需要半数以上的节点同意才能执行

### 注意事项

1、transaction log

      不要将此 日志路径 设置在一个 busy device上，会严重影响性能，因为每次的请求处理数据同步都需要进行记录此日志

2、Java heap size

      确保 RAM 足够， 避免内存与磁盘空间的交换的情况；通常在一个物理内存为4G的机器上，最多设置-Xmx为3G；最好 进行 压测

3、定时清理备份数据目录

4、log4j的配置：ROLLINGFILE

5、不要存储过多的信息在zookeeper

6、不要在同一个znode注册过多的watcher，每个占大约300kb，性能影响

7、不要强依赖zookeeper，常用 wchs，wchc，wchp这些命令查看watches的信息

 

### 总结

1、类文件系统结构，有路径节点，节点有值，有多版本，可设置节点ACL

2、只要大多数实例正常，则整个集群对外就可提供正常服务

3、leader选举，follower投票，observer不参与

4、适合读多写少的场景

5、zookeeper本身提供的API比较丑陋，推荐 http://curator.apache.org/

6、bookkeeper 记录日志流的系统

 

### 参考：

1、从paxos到Zookeeper分布式一致性原理与实践

2、http://zookeeper.apache.org/doc/r3.4.12/zookeeperAdmin.html

3、Paxos算法
