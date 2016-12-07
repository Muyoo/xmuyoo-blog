title: Spark学习笔记记录点
date: 2015-09-08 14:18:50
updated: 2015-09-08 14:18:50
tags:
- Spark
- Distributed System
categories:
- 技术
- Spark
---
![小时候幻想变成小人在草丛世界里探险，微距镜头让我的幻想落在画中 - By Muyoo](http://7vzs9m.com1.z0.glb.clouddn.com/练习5.jpg)

### Overview

1.	有两种对象：

	RDD（Resillient Distributed Dataset），弹性分布式数据集，可以存储任何形式的数据；
	
	共享变量（Shared Variables），可以在不同的集群任务中共享，提供两种共享变量：
	* boardcast variable：在不同任务之间共享的变量，只读
	* accumulators：在不同任务之间共享的计数器，可用于求和
<!-- more -->
	
### 连接Spark集群
1.	Java

	```Java
	SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
	JavaSparkContext sc = new JavaSparkContext(conf);
	```

	第一行是创建Spark的集群信息对象，第二行是连接集群；任何一个Spark的应用都通过一个Context对象来连接集群：

	> The first thing a Spark program must do is to create a JavaSparkContext object, which tells Spark how to access a cluster. To create a SparkContext you first need to build a SparkConf object that contains information about your application.
	
### RDD(Resillient Distributed Dataset)
RDD是一个可容错、可并行处理的数据集合。

1.	创建RDD：

	有两种方式来创建RDD：创建并行处理集合 和 使用外部数据。
	
	并行数据集合（Parallelized Collections）：由我们自己的程序driver program生成，Java中通过JavaSparkContext对象的parallelize方法指定：
	
	```Java
    List<Integer> data = Arrays.asList(1, 2, 3, 4, 5);
    JavaRDD<Integer> distData = sc.parallelize(data);
	```
	parallelize方法可以传入分区参数，告诉Spark将输入数据分成多少个分区。
	
	外部数据集（External Datasets）：任务Hadoop格式的数据，如HDFS、HBase，甚至可以是AWS S3。通过SparkContext对象的textFile方法指定：
	
	```Java
	JavaRDD<String> distFile = sc.textFile("data.txt");
	```
	外部数据集可以是一般的TextFiles、SequenceFile，或是Hadoop支持的其它任何格式的数据。外部数据集的输入可以直接指定目录、包含通配符的文件名，也支持直接指定gz文件。
	
	外部数据集同样支持指定分区，默认是一个Hadoop的block对应一个分区。
	
	* 详细外部数据集说明信息参见（主要是note部分）：[External Datasets](http://spark.apache.org/docs/latest/programming-guide.html#external-datasets)*
	
2.	RDD操作：transformations 和 actions

	- transformations: 从一个已有数据集创建出一个新的数据集（理解为原数据集的变换）

	> create a new dataset from an existing one
		
	transformations的操作在创建时并不会马上执行，只有在调用actions的方法时才会开始执行。
	- actions: 对数据集做处理，例如统计计数等，返回一个值
		
	> return a value to the driver program after running a computation on the dataset
		
	- 每个经过transform的RDD都可以重复使用：通过 persist 方法。
	- 官方代码示例：

    ```Java
    JavaRDD<String> lines = sc.textFile("data.txt"); //use an external dataset
    
    // laziness, it will bot be computed here
    JavaRDD<Integer> lineLengths = lines.map(s -> s.length());
    
    // cache the transformed lineLengths RDD, declear before use reduce method
    lineLengths.persist();
    
    // actions
    int totalLength = lineLengths.reduce((a, b) -> a + b);
    ```

	- transformation 的方法：[Functions of Transformations](http://spark.apache.org/docs/latest/programming-guide.html#transformations)
	- actions 的方法：[Functions of Actions](http://spark.apache.org/docs/latest/programming-guide.html#actions)
    - shuffle operations
    shuffle operations是一种很消耗资源的操作，官方并不推荐使用这种方式；
    
3. Java代码使用方式参考：[Passing Functions to Spark](http://spark.apache.org/docs/latest/programming-guide.html#passing-functions-to-spark)


