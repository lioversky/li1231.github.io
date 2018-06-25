---
title: Spark Structured Streaming Source Sink整理
date: 2018-06-22 08:35:32
tags: spark
categories: 技术
---

## Source源码调用

![](/images/s_source.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 
Structured Streaming在Source阶段的调用过程如上图
<!-- more -->
1.在start时会启动StreamExecution内部属性microBatchThread线程，在线程内部调用runBatches方法；
2.在方法内执行triggerExecutor.execute调用runBatch方法；
3.调用source的getBatch返回具体数据的DataFrame；
4.在各实现Source的类中，获取对应的流数据，以kafkaSource为例，在getBatch中传入start和end的offset参数，通过kafka metadata，获取各topic的parititon在当前时间要获取的offsetRanges，此工作在driver内执行，然后生成KafkaSourceRDD，传入kafka连接参数和offsetRanges等；
5.在RDD的compute方法内，首先调用getOrCreate方法获取CachedKafkaConsumer，并修正offsetRange，生成NextIterator迭代器；
6.在迭代器内调用CachedKafkaConsumer的get方法获取ConsumerRecord，在get内调用fetchData方法，此工作在各executor中执行，在 ConsumerRecord内保存着提前从kafka拉取出来的数据fetchedData，数据都是从其内部获取，当fetchedData为空时，调用kafkaConsumer的poll拉取数据填充；不为空拿到record并进行一系列fail的offset判断，正确后返回ConsumerRecord。


## Sink源码调用

![](/images/s_sink.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Structured Streaming在Sink阶段的调用过程如上图

1.前两步与Source阶段相同，在调用getBatch方法得到dataframe后，调用Sink的addBatch方法；
2.仍然以KafkaSink为例，在addBatch方法内调用KafkaWriter的write方法；
3.调用RDD的foreachPartition方法，得到iter后在各个executor中生成KafkaWriteTask执行execute方法；
4.通过CachedKafkaProducer.getOrCreate获取producer，在row中获取topic、key、value值，发送。