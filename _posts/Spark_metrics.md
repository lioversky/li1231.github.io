---
title: Spark metrics整理
date: 2018-06-22 17:28:32
tags: spark
categories: 技术
---

# 概述


spark使用metrics的包路径为：org.apache.spark.metrics，核心类：MetricsSystem。可以把Spark Metrics的信息报告到各种各样的Sink，比如HTTP、JMX以及CSV文件。

<!-- more -->
Spark的Metrics系统目前支持以下的实例：
		○ master：Spark standalone模式的master进程；
		○ worker：Spark standalone模式的worker进程；
		○ executor：Spark executor；
		○ driver：Spark driver进程；
		○ applications：master进程里的一个组件，为各种应用作汇报。

metrics结构主要分为sink和source，source指数据来源，Source内属性包含name和MetricRegistry，主要有：MasterSource、WorkerSource、ApplicationSource、StreamingSource、BlockManagerSource等。

![](/images/m_source.png)

sink是将指标发送到哪，目前有：ConsoleSink、GangliaSink、GraphiteSink、MetricsServlet、JmxSink、CsvSink

![](/images/m_sink.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 配置

在spark的conf目录下metrics.properties文件中配置，默认使用此文件，也可以通过-Dspark.metrics.conf=xxx指定。

## sink配置参数说明
		
##### ConsoleSink 是记录Metrics信息到Console中
|名称|默认值|描述|
|:-------------:|:-------------:|:-------------:|
|class|org.apache.spark.metrics.sink.ConsoleSink|Sink类
|period|10|轮询间隔|
|unit|seconds|轮询间隔的单位|
##### CSVSink  定期的把Metrics信息导出到CSV文件中
|名称|默认值|描述|
|:-------------:|:-------------:|:-------------:|
|class|org.apache.spark.metrics.sink.CsvSink|Sink类|
|period|10|轮询间隔|
|unit|seconds|轮询间隔的单位|
|directory|/tmp|CSV文件存储的位置|

##### JmxSink 可以通过JMX方式访问Mertics信息
|名称|默认值|描述|
|:-------------:|:-------------:|:-------------:|
|class|org.apache.spark.metrics.sink.JmxSink|Sink类|
		
##### MetricsServlet
|名称|默认值|描述|
|:-------------:|:-------------:|:-------------:|
|class|org.apache.spark.metrics.sink.MetricsServlet|Sink类|
|path|VARIES*|Path prefix from the web server root|
|sample|false|Whether to show entire set of samples for histograms ('false' or 'true') ｜|
		
这个在Spark中默认就开启了，我们可以在4040端口页面的URL后面加上/metrics/json查看
		
##### GraphiteSink
|名称|默认值|描述|
|:-------------:|:-------------:|:-------------:|
|class|org.apache.spark.metrics.sink.GraphiteSink|Sink类|
|host|NONE|Graphite服务器主机名|
|port|NONE|Graphite服务器端口|
|period|10|轮询间隔|
|unit|seconds|轮询间隔的单位|
|prefix|EMPTY STRING|Prefix to prepend to metric name|
		
##### GangliaSink 由于Licene的限制，默认没有放到默认的build里面，如果需要使用，需要自己编译
|名称|默认值|描述|
|:-------------:|:-------------:|:-------------:|
|class|org.apache.spark.metrics.sink.GangliaSink|Sink类|
|host|NONE|Ganglia 服务器的主机名或multicast group|
|port|NONE|Ganglia服务器的端口|
|period|10|轮询间隔|
|unit|seconds|轮询间隔的单位|
|ttl|1|TTL of messages sent by Ganglia|
|mode|multicast|Ganglia网络模式('unicast' or 'multicast')|

## sink配置样例

语法 syntax: [instance].sink.[name].[options]=[value]

		*.sink.console.class=org.apache.spark.metrics.sink.ConsoleSink
		*.sink.console.period=10
		*.sink.console.unit=seconds
以上为所有实例开启ConsoleSink，包括master、worker、dirver、executor

		master.sink.console.class=org.apache.spark.metrics.sink.ConsoleSink
		master.sink.console.period=10
		master.sink.console.unit=seconds

以上只为master开启ConsoleSink

## source配置样例
		
语法 syntax: [instance].source.[name].[options]=[value]

`master.source.jvm.class=org.apache.spark.metrics.source.JvmSource`
以上为master开启JvmSource

# 代码流程

1.各实例master、worker、dirver、executor在启动时创建MetricsSystem实例，在start时调用registerSources()、registerSinks()；
2.registerSources中在配置文件中找到对应实例的配置，获取Source的具体包名类名后，通过反射生成实例，再调用registerSource注册到registry中。各source中已定义好要记录的指标，可以继承Source采集自定义信息；
3.registerSinks中同样获取对应实例的配置，得到sink的包名类名反射生成实例，加到sinks列表中。
4.report方法中可调用所有sink的report方法（只在stop中调用）。


参考：

[http://hao.jobbole.com/metrics/](http://hao.jobbole.com/metrics/)
[https://www.jianshu.com/p/e4f70ddbc287](https://www.jianshu.com/p/e4f70ddbc287)
[https://www.iteblog.com/archives/1341.html](https://www.iteblog.com/archives/1341.html)
