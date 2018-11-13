---
title: Spark Streaming Source Kafka 0.10
date: 2018-07-31 10:00:00
tags: spark
categories: 技术
---

## 描述

Kafka从0.9版本后，取消了simple level和High level的api，统一了Consumer，所以spark streaming也只保留了Direct模式。Kafka的具体变化见kafka官方文档，本文只描述spark streaming集成kafka0.10版本，代码样例见[spark kafka样例](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)。本文档描述与实际可能有偏差，请注意甄别。


## 任务生成

通过KafkaUtils生成的是DirectKafkaInputDStream，在driver端会使用ConsumerStrategy创建KafkaConsumer实例，代码如下：

```scala
@transient private var kc: Consumer[K, V] = null
  def consumer(): Consumer[K, V] = this.synchronized {
    if (null == kc) {
      kc = consumerStrategy.onStart(currentOffsets.mapValues(l => new java.lang.Long(l)).asJava)
    }
    kc
  }
```

在start方法中读取出当前consumer的所有topic partition的offset，代码如下：

```scala
override def start(): Unit = {
    val c = consumer
    paranoidPoll(c)
    if (currentOffsets.isEmpty) {
      currentOffsets = c.assignment().asScala.map { tp =>
        tp -> c.position(tp)
      }.toMap
    }

    // don't actually want to consume any messages, so pause all partitions
    c.pause(currentOffsets.keySet.asJava)
  }
```  
由于目前对kafka client研究不是太透彻，猜测要想获取当前offset调用consumer的assignment方法时，如果不先pull一下，获取结果为空，但又怕pull时真取出来数据更新了offset位置导致丢失数据，所以paranoidPoll方法就是为了做修正的。

在生成batch时会调用DStream的compute方法，在DirectKafkaInputDStream的compute中，首先计算当前batch的各partition的起始位置，就是先获取各partition的当前最大offset，与当前处理的offset的差和每个限流的partition读取最大数据取最小。获取当前最大offset代码如下：

```scala
  protected def latestOffsets(): Map[TopicPartition, Long] = {
    val c = consumer
    paranoidPoll(c)
    val parts = c.assignment().asScala

    // make sure new partitions are reflected in currentOffsets
    val newPartitions = parts.diff(currentOffsets.keySet)
    // position for new partitions determined by auto.offset.reset if no commit
    currentOffsets = currentOffsets ++ newPartitions.map(tp => tp -> c.position(tp)).toMap
    // don't want to consume messages, so pause
    c.pause(newPartitions.asJava)
    // find latest available offsets
    c.seekToEnd(currentOffsets.keySet.asJava)
    parts.map(tp => tp -> c.position(tp)).toMap
  }
```

kafka consumer cliet 提供了一系列seek方法，seek(),seekToBeginning(),seekToEnd()，分别是跳到指定位置、最小和最大。上面代码是将当前consumer的offset跳到最大然后获取当前c.position，经测试此方法会将当前groupId在kafka保存的offset同时更新到最大，如果猜测正确，会像kafka 0.8版本那样，一旦有任务堆积程序异常终止后，由于有未完成的batch导致数据丢失情况。

## 任务计算

在Dstream生成KafkaRDD，执行任务时调用compute方法，spark把kafka consumer封装使用CachedKafkaConsumer，在获取数据时只需调用 `val r = consumer.get(requestOffset, pollTimeout)`，CachedKafkaConsumer只在executor中创建，使用`cache: ju.LinkedHashMap[CacheKey, CachedKafkaConsumer[_, _]]`缓存consumer实例，key的结构为`case class CacheKey(groupId: String, topic: String, partition: Int)`。

是否使用Cache Consumer由参数"spark.streaming.kafka.consumer.cache.enabled"控制，默认为true使用。在get方法时传入当前要获取的offset，get方法代码如下：

```scala
  def get(offset: Long, timeout: Long): ConsumerRecord[K, V] = {
    logDebug(s"Get $groupId $topic $partition nextOffset $nextOffset requested $offset")
    if (offset != nextOffset) {
      logInfo(s"Initial fetch for $groupId $topic $partition $offset")
      seek(offset)
      poll(timeout)
    }

    if (!buffer.hasNext()) { poll(timeout) }
    require(buffer.hasNext(),
      s"Failed to get records for $groupId $topic $partition $offset after polling for $timeout")
    var record = buffer.next()

    if (record.offset != offset) {
      logInfo(s"Buffer miss for $groupId $topic $partition $offset")
      seek(offset)
      poll(timeout)
      require(buffer.hasNext(),
        s"Failed to get records for $groupId $topic $partition $offset after polling for $timeout")
      record = buffer.next()
      require(record.offset == offset,
        s"Got wrong record for $groupId $topic $partition even after seeking to offset $offset " +
          s"got offset ${record.offset} instead. If this is a compacted topic, consider enabling " +
          "spark.streaming.kafka.allowNonConsecutiveOffsets"
      )
    }

    nextOffset = offset + 1
    record
  }
```

poll方法为实际调用kafka consumer的poll方法，从kafka读取的数据保存在buffer中，每次获取一条并与得到的消息比对offset，当不一样时，重置为要获取offset，完成后更新nextOffset。

## 并发问题

spark driect模式每个batch的task数与读取partition数对应，无法像receiver模式控制task数量，当core数远大于task数时会资源浪费，spark.streaming.concurrentJobs控制并发数，默认为1，当修改此值大于1时会出现"java.util.ConcurrentModificationException: KafkaConsumer is not safe for multi-threaded access"异常，因为每个KafkaConsumer实例确保同每个group的partitin只能有一个线程消费，在使用Cache consumer模式时，每个groupId的每个partition只有一个实例，而开启并发并且有batch并发时，会同时产生多线程消费数据，产生该错误。

可以不使用Cache模式解决并发问题，即"spark.streaming.kafka.consumer.cache.enabled"置为false，则每次消费每个线程都重新创建Consumer实例，在使用完自动close，不会出现上述错误。

