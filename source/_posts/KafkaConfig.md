title: Kafka 配置项
comments: true
date: 2015-10-22 17:26:25
updated: 2015-10-22 17:26:25
tags:
- Kafka
- Messaging System
- Config
categories:
- 技术
- Kafka
---
![世界尽头的晨曦让我这么怀念 在冰岛北部，几乎走到哪里都可以见到这座美丽的火山 - By Muyoo](http://7vzs9m.com1.z0.glb.clouddn.com/照片133.jpg)

### Kafka 部分配置项
本文只记录一些目前使用Kafka可能会比较有用的配置项。关于单个broker的，和关于topic-level的。其中有些配置项可以是topic级别的。
Kafka的版本是0.8.2.1。

- log.flush.interval：默认500，数据积累的量，超过这个量就将数据写到磁盘
- log.default.flush.interval.ms：数据写到磁盘的间隔时间，毫秒。默认与log.default.flush.scheduler.interval.ms相同，3000。
- log.retention.hours：数据保留的时间，小时。默认168
- log.segment.bytes：数据文件大小，默认是1024*1024*1024
- log.roll.{ms,hours}：数据文件切分的时间间隔，与大小限制同时起作用的，默认是24*7小时
- log.cleanup.policy：处理切分后数据的方式，delete或者compact。如果是delete，被切分出来的数据会被删除；如果是compact，切分出的数据会被压缩用于淘汰过期的数据。
- log.cleaner.min.cleanable.ratio：
- min.insync.replicas：
- log.roll.jitter.{ms,hours}：

#### Topic-Level
- topic.flush.intervals.ms：覆盖log.default.flush.interval.ms。
- topic.log.retention.hours：覆盖log.retention.hours
- topic.partition.count.map：覆盖创建topic时指定的partition数量。
- topic.log.segment.bytes：覆盖log.segment.bytes
- topic.log.roll.{ms,hours}：覆盖log.roll.{ms,hours}
- topic.log.cleanup.policy：覆盖log.cleanup.policy
- min.cleanable.dirty.ratio：覆盖log.cleaner.min.cleanable.ratio
- min.insync.replicas：覆盖min.insync.replicas
- segment.jitter.ms：覆盖log.roll.jitter.{ms,hours}

官方的topic level配置项：[topic level](http://kafka.apache.org/documentation.html#topic-config)
