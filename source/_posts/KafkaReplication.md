title: Kafka Replication
comments: true
date: 2015-10-16 16:23:44
updated: 2015-10-16 16:23:44
tags:
- Kafka
- Messaging System
categories:
- 技术
- Kafka
---
![ 还是比较喜欢水果风味较亮的咖啡，中培的曼特宁喝起来竟会联想到原始森林T.T - By Muyoo](http://7vzs9m.com1.z0.glb.clouddn.com/冰滴4.jpg)

上一篇文章简单记录了Kafka的基本工作框架、数据结构、Producer和Consumer。现在继续啃Kafka的官方文档，主要记录下Kafka中的复制集。
（Kafka版本 0.8.2.1）
<!-- more -->

### Replication的结构
Kafka中通过配置项来指定topic中partition的replication数量。（通常replication面临的问题：通常slave节点的使用率不高，数据间的传输备份会与提升吞吐量造成冲突；同时需要配置的项目也是比较繁琐。）
Kafka备份的数据单元是partition，每个partition有一个leader和若干follower。replica的总数取决于所有broker的数量（包括leader）。broker作为工作单元（服务器），一个partition可以有多个broker，反之一个broker下也有多个partition（partition可能是不同topic的）。通常情况下，partition的数量比broker多，而leader也是在这些broker中选出。

### 同步数据
follower同步从leader同步数据的方式与正常的Consumer相同，从leader读取数据然后存入自己的log队列。一条新的数据进入后，在所有“in sync”的replica成功将其放入自己的队列后，这条数据才能被正常的Consumer读取。这个过程在Kafka中被称为“committed”。
而“in sync”是指，leader会维护一个满足以下两个条件的replica节点集合：
- 节点必须能与zookeeper保持正常通信
- 它必须保证从leader同步的数据不能掉队太多
当如果有follower发生异常“dies”、“gets stuck”、“falls behind”时，leader会将其从“in sync”的集合中去掉。（关于“get stuck”的定义配置通过replica.lag.time.max.ms控制，关于“falls behind”的定义配置通过replica.lag.max.messages控制。）
