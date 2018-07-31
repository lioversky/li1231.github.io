---
title: Spark Streaming Source Kafka 0.8.2
date: 2018-06-22 08:35:32
tags: spark
categories: 技术
---

# 描述

针对kafka0.8.2的API，spark Streaming有两个版本的Source，Receiver和DirectAPI，其中Receiver对应为KafkaInputDStream继承自ReceiverInputDStream -> InputDStream，DirectAPI对应DirectKafkaInputDStream直接继承自InputDStream。相信此二者的具体差别网上有更详细的描述，在此简单介绍一下：

* Receiver是预拉取数据，即先把数据读取到各executor的内存中，由BolckManager分配管理；而DirectAPI在每个batch中只是生成对应metadata，记录要处理的fromOffset和untilOffset，在实际执行时才摘取数据；
* Receiver生成的task数与生成block数相关，而block数与spark.streaming.blockInterval有关；而DirectAPI的task数与kafka partition数一一对应；


正由于以上差别，在不同情景两种方式各有优劣，简单概括如下：

## 两种方式对比

### Receiver优势

1.	由于两种方式kafka底层实现的不同，Receiver不需要自己管理offset而DirectAPI需要自己管理；
2. 当消费的topic过多或者topic对应的partition过多时，DirectAPI会产生特别多的task，这会使spark对调度产生更大的开销；而Receiver可能通过spark参数控制；
3. DirectAPI在生成Batch时，会首先查询topic的metadata数据，再根据topicAndPartition获取当前endOffset来决定Batch的起止offset，如果kafka的partition异常如topic对应的partition无leader，会无法生成新batch导致spark程序失败，而Receiver模式由于事先建立连接，不会出现此种错误；

### DirectAPI优势

1. 由于Receiver是预拉取，并且定期更新consumer的offset，所以在有数据堆积时程序出错结束后会丢失数据，而DirectAPI不会丢失数据，可能会有数据重复消费情况；
2. spark在2.0以前，在每个batch结束之后会调用cleanMetadata方法，此方法会清除当前batch及之前的所有数据，包括metadta和block，所以在开启并行（spark.streaming.concurrentJobs>1）如果出现后面的batch先执行完，出现 block not found的异常，大部分都是由于此问题导致；
3. Receiver模式在kafka不太稳定情况下，日志经常会出现reblance，并且会出现数据重复消费情况；

# 实现原理与使用

## Receiver模式


## DirectAPI模式


