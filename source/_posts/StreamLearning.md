title: Stream Processing学习笔记
comments: false
date: 2015-10-12 10:57:50
updated: 2015-10-12 10:57:50
tags: 
- StreamProcessing
categories:
- 技术
- Kafka
---
![ 最喜在幽然清风下静心学习，青稞真的跟麦子很像 - By Muyoo](http://7vzs9m.com1.z0.glb.clouddn.com/藏地旅行205.jpg)

最近在学习实时流式数据处理的相关知识。这篇文章就作为学习笔记来整理、记录，所谓“好记性不如烂笔头”。

学习的资料不是某本书，也不是某某教程，对于这样概念性和实践性都比较强、同时又主要是依赖于一些组件工具的东西，只需要在网络上搜索相关的技术文档就好了。因此，我学习的材料是Samza的官方文档。Samza是一个很年轻的组件，依赖Kafka。我并非要使用它，而是它的官方文档不仅写了使用Samza的相关信息，还写了关于Stream Processing的概念。
<!-- more -->

### Stream Processing概念

Stream Processing依赖于消息系统（Messaging System）。先简单说下什么是messaging system：

> A messaging system is a fairly low-level piece of infrastructure—it stores messages and waits for consumers to consume them. -- Samza

消息系统即是一个将消息数据缓存在消息队列中，等待消息处理程序去取数据的系统。

而Stream Processing是建立在消息系统之上的一个数据处理概念：

> Stream processing is a higher level of abstraction on top of messaging systems, and it’s meant to address precisely this category of problems. -- Samza

Stream Processing建立在消息系统之上，可能一个消息系统也可能多个；同时，在Stream Processing过程中包括如何处理消息系统中若干问题。

### Stream Processing的常见问题

由于Stream Processing是建立在消息系统这样的基础设施之上的，所以消息系统本身的问题同时也是Stream Processing的问题。

1. 如果数据的消费程序挂掉、当前数据丢掉了，怎么处理？丢掉的数据如何恢复？
2. 重启的程序应该从什么位置继续工作？
3. 如果依赖的消息系统出现数据重发、数据漏发，怎么处理？
4. 如何对消息数据进行分组处理？
5. 消息数据处理进程的状态如何维护？

### Samza
#### Samza的Job、partition和task
下面的笔记关于Samza的工作模式，其中的方式可以借鉴到一般的Stream Processing中。
Samza的一个job按数据管理方式会被分成若干个partition，按工作进程会被分成若干个task。

数据以Kafka 的方式按Key被分到不同的partition中，相同key的数据一定会被分到同一个partition中。
![from http://samza.apache.org](http://7vzs9m.com1.z0.glb.clouddn.com/stream-partition.png)

Task是job中具体执行数据处理的工作单元，一个task不局限于某一个job，它可以接收多个输入流，也就是说一个task可以从多个partition上读取数据；但反过来，一个partition的数据只能由一个task读取（从某一个task角度看，是1对N的关系；从某一个partition角度看，是1对1的关系。），因此partition的数量不能超过task的数量（有可能会出现task没有对应partition的情况）。多个task可以运行在多台机器上，其中资源的调度是通过yarn分配的。
![from http://samza.apache.org](http://7vzs9m.com1.z0.glb.clouddn.com/steam-task.png)

在Samza中的数据流与job之间可以形成一个数据流图（Dataflow Graphs），如图：
![from http://samza.apache.org](http://7vzs9m.com1.z0.glb.clouddn.com/dataflow-graph.png)
这样的好处是不同的job不必都写在同一份代码中；同时在这种方式下，对下游job进行操作时不影响上游job。
> The jobs are otherwise totally decoupled: they need not be implemented in the same code base, and adding, removing, or restarting a downstream job will not impact an upstream job. -- Samza

