title: Kafka 水平扩展、集群镜像
comments: true
date: 2015-10-20 10:43:13
updated: 2015-10-20 10:43:13
tags:
- Kafka
- Messaging System
categories:
- 技术
- Kafka
---
![日出总是让人心中充满无限希望与感动。站在长江源头看日出~ - By Muyoo ](http://7vzs9m.com1.z0.glb.clouddn.com/藏地旅行68.jpg)

之前阅读了Kafka的基本概念和关于Replica的文档，现在已是晕晕乎乎。在这大好的清晨微风徐徐中，继续阅读关于Kafka集群镜像的内容。
（Kafka版本：0.8.2.1）
<!-- more -->

### Kafka集群镜像
Kafka的集群镜像与Replica不同。Replica作用在单个集群中不同的节点之间；而建立集群镜像在不同的Kafka集群之间发生作用。在集群镜像的过程中，源数据来自于一个或多个不同的Kafka集群，然后数据流向一个目标集群：
![from Apache Kafka](http://7vzs9m.com1.z0.glb.clouddn.com/kafka-mirror-maker.png)
Kafka提供了专用的工具实现创建集群镜像。

这个操作的一个用途之一是创建数据中心的备份。但在严格意义上讲，它不完全是“备份”。因为创建后的镜像与源集群在partition数量、offset方面都会有所不同。但由于其使用的message key仍然不变并且也是通过Consumer的方式读取数据的，所以其顺序仍然相同。

### 扩展Kafka集群
添加新broker：为新的broker分配一个唯一id，然后在新的服务器上启动Kafka即可。
做到这一步完成新broker的添加，但它只有在创建新的topic时才会参与工作。除非将已有的partition迁移到新的服务器上面。

迁移数据到新broker：迁移数据只能手动执行，但整个过程是自动完成的。Kafka先将新的Server设置为要迁移的partition的follower，让它进行replica。等它将目标partition的数据完全复制，且这个新的server已经进入in sync的列表后，已有的一个replica就会删除自己的partition数据。
Kafka提供kafka-reassign-partitions.sh工具做添加新broker后的数据迁移。

#### kafka-reassign-partitions.sh 使用方法
本文记录下笔者的尝试过程，详细的文档说明可以在这里找到：[kafka-reassign-partitions.sh](http://kafka.apache.org/documentation.html#basic_ops_cluster_expansion)
依据官方的文档，kafka-reassign-partitions.sh是用来重新分配partition和replica到broker上的工具。简单实现重新分配需要三步：
- 生成分配计划（generate）
- 执行分配（execute）
- 检查分配的状态（verify）

在我原有的集群中，只有一个broker，broker.id = 0；现在我添加了一个broker到集群中，broker.id = 1。下面记录如何对原有的partition进行重新分配。
添加broker很简单，只需要在新的机器上拷贝一份新的server.properties，然后改一个新的broker.id，启动即可。

1. 生成分配计划
```
./bin/kafka-reassign-partitions.sh --zookeeper 127.0.0.1:2181 --topics-to-move-json-file topics-to-move.json --broker-list "1" --generate
```
其中，topic-to-move.json 是事先自己写好的一个json文件，里面包含要进行迁移的topic。例如我用来实验的是这样：
```
{"topics":
        [{"topic":"test"}],
            "version": 1
}
```
用 kafka-topics.sh 工具查看topic信息是：
```
Topic:test  PartitionCount:2    ReplicationFactor:1 Configs:
    Topic: test Partition: 0    Leader: 0   Replicas: 0 Isr: 0
    Topic: test Partition: 1    Leader: 0   Replicas: 0 Isr: 0

```
因为原有集群只有一个broker，所以两个partition都在broker 0上，而broker 0本身也是leader，在“in sync”的replica也都是0。
执行生成计划的命令以后，会出现类似文档里那样的信息，有两个json格式信息：
```
Current partition replica assignment

{"version":1,"partitions":[{"topic":"test","partition":1,"replicas":[0]},{"topic":"test","partition":0,"replicas":[0]}]}
Proposed partition reassignment configuration

{"version":1,"partitions":[{"topic":"test","partition":1,"replicas":[1]},{"topic":"test","partition":0,"replicas":[1]}]}
```
2. 执行分配
在生成计划后，可以按照计划来执行重新分配，执行命令：
```
./bin/kafka-reassign-partitions.sh --zookeeper 127.0.0.1:2181 --reassignment-json-file expand-cluster-reassignment.json --execute
```
在官方文档中，提供的例子只有这一行命令与其输入样式。然而，所需的 expand-cluster-reassignment.json 文件里面是什么确没有直接说明。（让我花了好多时间研究=。=）这个文件的内容就是：刚才执行--generate后，第！二！个！json！是的，将它直接粘贴到这个文件里保存就可以了。执行命令后的会出现如下的信息：
```
Current partition replica assignment

{"version":1,"partitions":[{"topic":"test","partition":1,"replicas":[0]},{"topic":"test","partition":0,"replicas":[0]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions {"version":1,"partitions":[{"topic":"test","partition":1,"replicas":[1]},{"topic":"test","partition":0,"replicas":[1]}]}
```
此时，Kafka就已经开始进行重新分配、数据迁移了。
3. 检查分配的状态
这一步很好操作。只需要执行命令就可以：
```
./bin/kafka-reassign-partitions.sh --zookeeper 127.0.0.1:2181 --reassignment-json-file expand-cluster-reassignment.json --verify
```
json文件依然是--execute时的那个。然后会出现如下信息：
```
Status of partition reassignment:
Reassignment of partition [test,1] completed successfully
Reassignment of partition [test,0] completed successfully
```
这样就完成了。用kafka-topics.sh工具查看一下当前topic信息就变成了这样：
```
Topic:test  PartitionCount:2    ReplicationFactor:1 Configs:
    Topic: test Partition: 0    Leader: 1   Replicas: 1 Isr: 1
    Topic: test Partition: 1    Leader: 1   Replicas: 1 Isr: 1
```

在这个例子中，我将原来在broker 0上的两个partition都重新分配到了broker 1上。其实可以将两个partition都分在这两个broker上。只需要在生成计划的那一步，指定--broker-list就可以了，将它指定为：--broker-list "0,1"。这个参数就是目标broker的id列表。例如我这样指定后topic的信息就变成了这样：
```
Topic:test  PartitionCount:2    ReplicationFactor:1 Configs:
    Topic: test Partition: 0    Leader: 0   Replicas: 0 Isr: 0
    Topic: test Partition: 1    Leader: 1   Replicas: 1 Isr: 1
```

#### kafka-topics.sh add partition 添加partition
到此我们可以为broker重新分配partition和replica。但是要增加partition怎么办？
我问了好久google，google也没有给我合适的答案，倒是告诉了我一个不幸的消息：kafka-add-partition.sh这样的工具只在0.8.0版本中才有。后续的版本都没了，都没了，没了，没…… Orz 那肿么办？！Kafka这样的神器不会傻到不支持增加partition吧？
于是就在0.8.2.1提供的工具中查找，发现kafka-topics.sh工具--partitions选项的说明是这样的：
```
--partitions <Integer: # of partitions> The number of partitions for the topic 
                                          being created or altered (WARNING:   
                                          If partitions are increased for a    
                                          topic that has a key, the partition  
                                          logic or ordering of the messages    
                                          will be affected                     
```
看到了吗？“the number of partitions for the topic being created or altered”，然后这个工具有一个选项就是 --alter 啊！试了一下，果然可以！
```
./bin/kafka-topics.sh --zookeeper 127.0.0.1:2181 --alter --topic test --partitions 4
```
执行后出现如下信息：
```
WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affected
Adding partitions succeeded!
```
再查看topic信息就变成了这样：
```
Topic:test  PartitionCount:4    ReplicationFactor:1 Configs:
    Topic: test Partition: 0    Leader: 0   Replicas: 0 Isr: 0
    Topic: test Partition: 1    Leader: 1   Replicas: 1 Isr: 1
    Topic: test Partition: 2    Leader: 1   Replicas: 1 Isr: 1
    Topic: test Partition: 3    Leader: 0   Replicas: 0 Isr: 0
```
但是注意，就如官方提示所言，在改变partition的数量后，如果消息数据进入partition是以key分配的，那么消息按key分配和消息的顺序都会受影响。
这样就添加了partition。

添加partition的这种方法再配合重新分配的方法，就可以当做一种水平扩展的方案。主要就是依赖这两个工具：
- kafka-topics.sh：添加partition
- kafka-reassign-partitions.sh：重新分配partition、迁移数据
