---
title: Spark stop过程
date: 2019-01-09 15:00:00
tags: spark
categories: 技术
---

## SparkContext关闭过程

SparkContext的stop方法可以在多个地方调用，如SparkSession的close和stop方法，StreamingContext的stop方法，SparkContext启动过程加的钩子。关闭过程如下：

<!-- more-->

1. 首先向所有注册的Listener必送`SparkListenerApplicationEnd`事件；
2. 关闭所有ui服务；
3. 调用MetricsSystem的report方法将所有指标发送出去；
4. 关闭清理线程ContextCleaner；
5. 关闭Executor 动态分配管理器
6. 关闭所有监听；
7. 关闭eventLogger；
8. 关闭dagScheduler，在dagSchedulerstop方法中会关闭taskScheduler，在TaskSchedulerImpl的stop方法中，如果backend不为空则调用backend.stop，在spark on yarn模式中，backend为CoarseGrainedSchedulerBackend实例，在stop中会通过rpc会向Executor发送StopExecutor事件，CoarseGrainedExecutorBackend接收StopExecutor事件后执行Executor的stop方法停止；
9. 关闭rpc；
10. 关闭SparkEnv，在SparkEnv的stop方法中会关闭一系列manager线程，包括blockManager、metricsSystem；

## StreamingContext关闭过程

如果是SparkStreaming应用，会先停止StreamingContext再停止SparkContext。关闭过程如下：

1. 关闭JobScheduler，如果优雅停止则等待当前任务全部执行完成再关闭DStreamGraph，否则直接停止；
2. 移除MetricsSystem的streamingSource，streamingSource主要负责记录streaming执行的指标信息；
3. 移除spark streaming的ui tab；
4. 如果需要关闭SparkContext，调用SparkContext的stop方法。


## Executor关闭过程

略


