---
title: Spark Streaming Source Kafka 0.8.2
date: 2018-07-31 10:00:00
tags: spark
categories: 技术
---

# 描述

针对kafka0.8.2的API，Spark Streaming有两个版本的Source，Receiver和DirectAPI，其中Receiver模式使用HighLevel对应为KafkaInputDStream，继承自ReceiverInputDStream再继承InputDStream，DirectAPI使用SimpleConsumer对应DirectKafkaInputDStream直接继承自InputDStream。此二者的具体差别网上有更详细的描述，在此简单介绍一下：
<!-- more -->
* Receiver是预拉取数据，即先把数据读取到各executor的内存中，由BolckManager分配管理；而DirectAPI在每个batch中只是生成对应metadata，记录要处理的fromOffset和untilOffset，在实际执行任务时才拉取数据；
* Receiver生成的task数与生成block数相关，block数与spark.streaming.blockInterval有关；而DirectAPI的task数与kafka partition数一一对应；


正由于以上差别，在不同情景两种方式各有优劣，简单概括如下：


### Receiver优势

1.	由于两种方式kafka底层实现的不同，Receiver不需要自己管理offset而DirectAPI需要自己管理；
2. 当消费的topic过多或者topic对应的partition过多时，DirectAPI会产生特别多的task，这会使spark对调度产生更大的开销；而Receiver可以通过spark参数控制；
3. DirectAPI在生成Batch时，会首先查询topic的metadata数据，再根据topicAndPartition获取当前endOffset来决定Batch的起止offset，如果kafka的partition异常如topic对应的partition无leader，会无法生成新batch导致spark程序失败或者连接非partitionLeader，而Receiver不会出现此种错误；

### DirectAPI优势

1. 由于Receiver是预拉取，并且可以控制在处理完成batch后更新consumer的offset，所以在有数据堆积时程序异常终止会丢失数据，而DirectAPI不会丢失数据，可能会有数据重复消费情况；
2. spark在2.0以前，在每个batch结束之后会调用cleanMetadata方法，此方法会清除当前batch及之前的所有数据，包括metadta和block，所以在开启并行（spark.streaming.concurrentJobs>1）如果后面的batch先执行完，会出现 block not found的异常，大部分都是由于此问题导致；
3. Receiver模式在kafka不太稳定情况下，日志经常会出现reblance，并且有数据重复消费情况；

# 实现原理与使用

## Receiver模式

Receiver模式实现类为KafkaInputDStream，继承自ReceiverInputDStream，实现其getReceiver方法。KafkaReceiver在onStart方法中创建指定线程数读取数据，再通过Receiver的store方法写到blockManager中；

1. 下面代码是创建KafkaInputDStream并生成numStreams个Receiver：

	```java
	List<JavaPairDStream<String, String>> kafkaStreams = new ArrayList<JavaPairDStream<String, String>>(numStreams);
	for (int i = 0; i < numStreams; i++) {
	  kafkaStreams.add(KafkaUtils.createStream(streamingContext,zookeeper, groupId, topicMap,storageLevel));
	}
	```

2. 下面代码负责为每个进程创建线程（源码）：

	```scala
	val topicMessageStreams = consumerConnector.createMessageStreams(
	      topics, keyDecoder, valueDecoder)
	val executorPool =
	      ThreadUtils.newDaemonFixedThreadPool(topics.values.sum, "KafkaMessageHandler")
	try {
	  // Start the messages handler for each partition
	  topicMessageStreams.values.foreach { streams =>
	    streams.foreach { stream => executorPool.submit(new MessageHandler(stream)) }
	  }
	} finally {
	  executorPool.shutdown() // Just causes threads to terminate after work is done
	}
	```

3. 下面代码在MessageHandler负责读取处理消息数据（源码）：

	```scala
	val streamIterator = stream.iterator()
	while (streamIterator.hasNext()) {
	  val msgAndMetadata = streamIterator.next()
	  store((msgAndMetadata.key, msgAndMetadata.message))
	}
	```

4. ReceiverInputDStream的compute创建BlockRDD（源码）：

	```scala
	//ask the tracker for all the blocks that have been allocated to this stream
	// for this batch
	val blockInfos = receiverTracker.getBlocksOfBatch(validTime).getOrElse(id, Seq.empty)
	
	// Create the BlockRDD
	createBlockRDD(validTime, blockInfos)
	```



## DirectAPI模式

direct模式需要自己保存offset，可以通过checkpoint保存或者外部存储。使用`KafkaUtils.createDirectStream`创建InputDStream，此方法有多个重载，根据需要使用。

1. 在每次启动时都要传入partitionOffset，offset从保存位置读取，创建stream代码如下：

	```java
	Map<TopicAndPartition, java.lang.Long> offsets = ....
	Class<Tuple2<String, String>> c = (Class<Tuple2<String, String>>) (Class) Tuple2.class;
	JavaInputDStream<Tuple2<String, String>> messages = KafkaUtils
        .createDirectStream(streamingContext, String.class,
            String.class, StringDecoder.class, StringDecoder.class, c, kafkaParams, offsets,
            new Function<MessageAndMetadata<String, String>, Tuple2<String, String>>() {
              public Tuple2<String, String> call(MessageAndMetadata<String, String> md)
                  throws Exception {
                return new Tuple2<String, String>(md.key(), md.message());
              }
            }
        );
	```
2. 在生成每个batch任务时，DirectKafkaInputDStream内保存当前已记录的offsets值，再通过获取latestLeaderOffsets，计算出本本次batch要处理的数据区间，此区间会被每个分区的配置最大消费数限制；

	```scala
	val untilOffsets = clamp(latestLeaderOffsets(maxRetries))
	```
3. 正是由于每个batch都要获取当前各paritition最大的offset值，`kc.getLatestLeaderOffsets(currentOffsets.keySet)`，所以在kafka的partition出现异常时会导致任务出错或者由于连接超时阻塞任务生成；
4. 得到currentOffsets和untilOffsets后，创建KafkaRDD，rdd内部属性offsetRanges记录此rdd要处理的各partition的offset区间值，通过此属性生成对应数量的KafkaRDDPartition。

	```scala
	val rdd = KafkaRDD[K, V, U, T, R](
      context.sparkContext, kafkaParams, currentOffsets, untilOffsets, messageHandler)
	```
	```scala
	val offsetRanges = fromOffsets.map { case (tp, fo) =>
        val uo = untilOffsets(tp)
        OffsetRange(tp.topic, tp.partition, fo, uo.offset)
    }.toArray
	```
	```scala
	override def getPartitions: Array[Partition] = {
	    offsetRanges.zipWithIndex.map { case (o, i) =>
	        val (host, port) = leaders(TopicAndPartition(o.topic, o.partition))
	        new KafkaRDDPartition(i, o.topic, o.partition, o.fromOffset, o.untilOffset, host, port)
	    }.toArray
 	}
	```
5. 在执行每个batch job时，为每个partition生成KafkaRDDIterator实例，根据每个partition中的信息创建SimpleConsumer连接，并认为此节点为对应kafka Partition的Leader，如果在此之前切换Leader也会出现kafka异常。在KafkaRDDIterator中通过getNext方法即时获取数据。


