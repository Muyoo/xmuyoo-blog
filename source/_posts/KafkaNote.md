title: Kafka 基本概念
comments: true
date: 2015-10-14 12:00:46
updated: 2015-10-14 12:00:46
tags: 
- Kafka
- Messaging System
categories:
- 技术
- Kafka
---
![ 每次旅行都能让我回味到下次旅行时，罗布林卡的花真漂亮 - By Muyoo](http://7vzs9m.com1.z0.glb.clouddn.com/藏地旅行125.jpg)
（Kafka版本 0.8.2.1）
### Kafka框架下的基本工作模式
Kafka是一个消息处理系统。从大面上看，产生消息数据的生产者（Producer）将数据推送到Kafka集群，Kafka集群将消息数据保留起来；然后消费消息数据的消费者（Consumer）向Kafka集群发起请求来消费数据。如图：
![from Apache Kafka](http://7vzs9m.com1.z0.glb.clouddn.com/kafka-producer_consumer.png)
<!-- more -->

### Kafka基本数据结构
消息数据在Kafka中先按照指定的名称进入一组消息队列，这组消息队列有统一的名称。在Kafka中，这样的消息队列组称为Topic。在每个Topic中有多个消息队列partition。每条消息进入Kafka后，按照key被分配到某个partition中，添加到队列的末尾。如下图：
![from Apache Kafka](http://7vzs9m.com1.z0.glb.clouddn.com/kafka-log_anatomy.png)
对每个partition中的每条数据，kafka会维护一个它在队列中的偏移量（可以理解为类似数组下标的东西）；consumer通过偏移量来读取指定的数据。
对于进入Kafka的数据，Kafka会保留一段时间；过了这段时间后，Kafka就会删除最早的一条数据，也就是从队列头的数据。（保留的时间长短可以通过配置文件设置。）在这个过程中，即使数据尚未被消费过也同样会被删除。同时，Kafka也因此会在设置的时间段内保留着所有进入Kafka集群的数据，不论数据是否被消费过。（所以在kafka中的数据可以被重复消费。）（Kafka删除数据的另一种方式是通过每个Partition的大小，partition文件超过设置的大小时就会删除旧数据。）
offset与这种数据保留方式的好处有两点：
- 保证数据占用的空间不会无限制增长；
- consumer在消费数据时，互相之间是独立的，不会产生影响。

### Kafka的分布式结构
Kafka的分布式结构体现在物理存储partition上。在物理存储方式上，一个partition被分成了若干份，分别存储在不同的机器上。每个partition的数据也会被复制若干份（数量可配置）。
每个partition会有一个“leader”的角色，负责处理所有读写请求；同时也有多个“follower”，当leader挂掉时，其中一个follower会承担起leader的角色，变成新的leader。

### Producer
Producer负责产生数据发送到Kafka中。Producer发送数据有两种方式，第一种是随机发送数据到集群中；第二种是以key来指明数据所属的topic与partition。推荐使用第二种方式，因为通过指定key可以将同一类数据发送到同一partition，后端的consumer就可以只针对这个partition来处理数据。
Producer采取异步发送数据方式。Producer发送数据时是批量发送，发送的周期可以以积累的数据大小和时间周期两个条件来同时配置。
Kafka中的数据可以进行压缩，有gzip和snapy两种格式。关于压缩参考：[Kafka数据压缩](https://cwiki.apache.org/confluence/display/KAFKA/Compression)。关于Producer发送数据的相关配置和API可参考：[配置项](http://kafka.apache.org/documentation.html#newproducerconfigs) 和 [API](http://kafka.apache.org/082/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)

在Producer端，对于数据保证没有做很多工作。Producer可以设置等待响应的时间，也可以设置就一直等到有响应为止。官方对这点的说法是对于数据生产者来说，它只需要做到在数据发送失败时能重发就可以了，对数据的保证（guarantee）的要求并非很强；并且也在考虑将来加入相关的机制。
Kafka中Producer的默认发送方式是“at-least-once”（可配置）。

### Consumer
Consumer负责从partition中消费数据。每个Consumer被组合成一个Consumer Group。Consumer Group中的每个Consumer可以消费不同partition的数据，但反过来partition只能被一个consumer消费，也因此实际消费的consumer数量不可以超过partition的数量。如下图：
![from Apache Kafka](http://7vzs9m.com1.z0.glb.clouddn.com/kafka-consumer-groups.png)
这种模式可以同时兼顾到消息处理模拟模型中的queue和publish-subscribe两种方式。另外，在传统的消息处理系统中，由于发送数据与消费数据是异步进行的，所以当消息系统将消息发送出去后，就不能保证数据在consumer一端仍然有序（理解是因为传统的系统中，后端有多个consumer去消费queue中的数据）。而Kafka中从partition到consumer是1对1的关系，所以能保证数据原有的顺序。
但是，这种顺序仅在partition中被保证，而无法保证数据在topic层面是有序的。

Consumer读取数据时需要向作为某个partition的leader的broker发送读取该partition数据请求，请求中指明数据的offset和读取的批量大小。
因为Consumer本质上是通过指定offset去读取数据的，那么再向Kafka做ack/commit offset是否还有必要？只需要Consumer自己记录下offset就可以进行读取？
答案是是的。但这仅限于Consumer永远不会挂掉的情况。否则就需要考虑怎么记录下Consumer的处理位置。

Consumer在请求数据时，采取长连接等待的方式，也就是直到有数据进入时才读取数据；也可以选择设置数据的大小，consumer就会等进入的数据达到设置的大小时才取回数据（这种方式可以确保一次性传输较大的数据，减少网络传输次数等）。
从consumer的角度看数据保证（guarantee）的问题。在consumer看来，因为它需要维护读取数据的位置，所以相比Producer而言会复杂一些。官方也对比了三种方式：
- 读数据 -> 存储位置 -> 处理数据：这种方式下，当在“处理数据”阶段挂掉时，由于已经保存了位置状态，当下次请求数据时就会从下一条数据开始。这样的话，数据就会丢失。这符合“at-most-once”的情况。
- 读数据 -> 处理数据 -> 存储位置：这种方式下，当consumer在“处理数据”后、“存储位置”前（或“存储位置”的过程中）挂掉时，数据的位置状态没有被记录下来；在下一次的过程中，这条数据就会重复处理一次。这符合“at-least-once”的情况。
- 读数据 -> 输出数据：这种处理方法与上述两种方法的角度、思路不同。这种方法是将offset存储和输出的数据存储在一起；上述两种方法是基于messaing system中关于数据发送来讨论（“at-most-once”，“at-least-once”，“exactly-once”）。官方的例子是，在他们的Hadoop ETL过程中，将offset和输出数据一起存在HDFS中，这样offset和数据要么一起更新，要么都不更新。
