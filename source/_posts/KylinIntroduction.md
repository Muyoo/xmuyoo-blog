title: Kylin Learning Note —— What? (Kylin, Cube)
comments: true
date: 2016-03-23 11:28:15
updated: 2016-03-23 11:28:15
tags:
- Kylin
- OLAP
- Cube
- Data Model
categories:
- 技术
- Kylin
---
![阳光下的星星 - By Muyoo](http://7vzs9m.com1.z0.glb.clouddn.com/2015-03-21%20164026.jpg)

## What is Kylin?

[Kylin](http://kylin.apache.org) is an [OLAP](https://en.wikipedia.org/wiki/Online_analytical_processing) system developed by the data team of eBay China.

Fast is the biggest selling point. It offers second level latency for query over 10 billions rows data. According to the official words, it can let you query big table in Hive at sub-second latency. Sounds cool, right?
<!-- more -->

Another important key point is its idea of processing data, that is pre-processing the big data. What does it mean? Kylin will create some data segments which contain middle level results before we querying. Therefore, Kylin uses extra disk spaces to exchange the low latency performance.
Kylin depends on:

- Hadoop Yarn
- HBase
- Hive
- Zookeeper

the versions above are according to the Kylin's version. It can be found on the doc pages of Kylin.

The third point is that Kylin is used for low latency statistic, not searching. If you want to search something like some users' details, unfortunately, in my opinion, Kylin may not be your best choice.

## Data Model of Kylin

The most important key word is ```Cube```. As you use Kylin, you will create a cube, build a cube, run cube jobs, merge cubes and so on. So, what is a cube? 

Before answering this question, let's see how to make a cube ready. To do this, we will do 4 steps:

- Load Hive tables into Kylin we need
- Create a data model
- Design the Cube
- Build cube

I'm not going to talk about the details about these steps. If you want to know more about them, you can find them in my another article. 

After we loading the hive tables into Kylin, we need to create a data model first. A data model is a description about what hive tables, dimensions (such as columns used for group by), measures (such as columns used for statistic) we need. It is not a real data block.
And then we need to create a cube description. It contains what dimensions we need, same as the model; and what measures we need, the difference from the model is that we design which statistic function we use here; and the time range. It is also not a data block yet, but it is called ```cube_desc``` and stored in HBase.
At last we build a cube according to the information created above two. At this step, it creates the real data segments on HDFS stored in HBase. So, in a view, we can say that the model and the cube are logical definitions and the data segment is a physical definition. A cube is consists of numbers of segments.

## References

- [Kylin Home Page](http://kylin.apache.org)
- [OLAP Wikipedia](https://en.wikipedia.org/wiki/Online_analytical_processing)

