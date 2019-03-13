---
title: Spark与Flink提交与执行对比
date: 2018-12-20 12:00:00
tags: spark flink
categories: 技术
---

本文主要讲述Spark与Flink在yarn上执行时的应用提交过程和执行方式的对比。

# spark on yarn 任务提交过程

### 命令生成

使用spark-submit提交命令，是重新调用了spark-class脚本

```shell
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```
<!-- more -->
在spark-class内部启动java进程org.apache.spark.launcher.Main生成最后的启动命令，更加详细文档可参考：[Apache Spark源码剖析-Shell](http://www.winseliu.com/blog/2016/05/08/rrc-apache-spark-source-inside-shell/)

### 提交任务

1. 脚本在本地启动主类`org.apache.spark.deploy.SparkSubmit`，根据参数生成childArgs, childClasspath, sparkConf, childMainClass等；
2. 如果是yarn client模式，反射调用用户通过--class自定义的主类方法，在生成SparkContext时通过`YarnClientSchedulerBackend.submitApplication`启动ApplicationMaster调用`registerAM`注册成为AM并向ResourceManager申请资源；

![park client模式](/images/spark_client.jpg)
<center><font size="2" color=gray>spark client模式</font></center>

3. 如果是cluster模式，SparkSubmit调用 `org.apache.spark.deploy.yarn.YarnClusterApplication` 启动 `org.apache.spark.deploy.yarn.Client.run` 方法，在submitApplication方法内生成yarnClient提交命令、上传所需要jar包等动作，并不执行用户代码。申请到的container中启动org.apache.spark.deploy.yarn.ApplicationMaster后调用run方法，在run执行`runDriver`方法，在`runDriver`中首先以独立线程启动用户main方法，然后调用`registerAM`注册成为AM并向ResourceManager申请资源；

![spark cluster模式](/images/spark_cluster.jpg)
<center><font size="2" color=gray>spark cluster模式</font></center>

5. 申请到资源后，在Container中启动CoarseGrainedExecutorBackend，来启动Executor。

### 提交命令

```shell
sudo -u hdfs /data0/workspace/spark-2.3.2/bin/spark-submit \
--name pipeline-test \
--conf "spark.serializer=org.apache.spark.serializer.KryoSerializer" \
--class com.XXX.PipelineMain \
--conf spark.streaming.blockInterval=100 \
--conf "spark.streaming.receiver.maxRate=100000" \
--num-executors 24 \
--executor-cores 4 \
--executor-memory 5G \
--driver-memory 3G \
--master yarn \
--deploy-mode cluster \
--queue back1 \
--conf "spark.sql.shuffle.partitions=20" \
--conf spark.scheduler.mode=FAIR \
--conf "spark.yarn.submit.waitAppCompletion=false" \
--conf spark.scheduler.listenerbus.eventqueue.capacity=30000 \
pipeline-spark-kafka-0-8-1.0-SNAPSHOT.jar \
pipeline-test
```



# Flink on yarn 任务提交过程

### client端执行过程

1. 在使用bin/flink脚本提交任务时，调用`org.apache.flink.client.cli.CliFrontend`类，在main方法中首先
然后获取全局配置信息：`GlobalConfiguration.loadConfiguration(configurationDirectory)` 再生成命令行解析器；然后根据参数执行具体方法，本次只关注run方法；

2. 在run方法中先解析参数，然后创建PackagedProgram，进入`runProgram(customCommandLine, commandLine, runOptions, program)`； 先创建ClusterDescriptor
`ClusterDescriptor<T> clusterDescriptor = customCommandLine.createClusterDescriptor(commandLine);`
生成子类实例org.apache.flink.yarn.YarnClusterDescriptor；在此处会根据是否传入-yd 判断Detach进入不同的代码段；

3. 如果为detach模式，先生成jobGraph，过程中调用 `org.apache.flink.client.program.PackagedProgramUtils.createJobGraph` 时，在此处分为两种方法执行：program entryPoint和interactive mode;interactive mode，在interactive模式下 `org.apache.flink.client.program.OptimizerPlanEnvironment.getOptimizedPlan(packagedProgram)` 中，通过 `org.apache.flink.client.program.PackagedProgram.invokeInteractiveModeForExecution()` 调用了自定义主类的main方法生成FlinkPlan；再调用 `clusterDescriptor.deployJobCluster(clusterSpecification,jobGraph,runOptions.getDetachedMode())` 提交jobGraph生成yarn job,最后关闭ClusterClient；

4. 如果为非detach模式，clusterDescriptor.deploySessionCluster(clusterSpecification)中生成am并启动YarnSessionClusterEntrypoint，完成后返回RestClusterClient实例；然后进入`executeProgram`方法，执行`org.apache.flink.client.program.restRestClusterClient的.run`方法，在此处分为两种方法执行：program entryPoint和interactive mode;interactive mode模式是通过`prog.invokeInteractiveModeForExecution()`反射自定义主类的`main`方法执行，由 `org.apache.flink.streaming.api.environment.StreamContextEnvironment.execute`调用`ClusterClient.run` 方法最后执行 `org.apache.flink.yarn.RestClusterClient.submitJob(JobGraph)` 提交任务。

### jobmanager端非detched模式

1. 在am中的主类为YarnSessionClusterEntrypoint，在main方法中通过 `ClusterEntrypoint.runClusterEntrypoint(yarnSessionClusterEntrypoint)` 调用到`startCluster`方法，再进入到`runCluster`，在此方法内首先由`initializeServices`创建所有本地服务实例,然后创建DispatcherResourceManagerComponent实例，代码如下：

	```java
	final DispatcherResourceManagerComponentFactory<?> dispatcherResourceManagerComponentFactory = createDispatcherResourceManagerComponentFactory(configuration);
	clusterComponent = dispatcherResourceManagerComponentFactory.create(
		configuration,
		commonRpcService,
		haServices,
		blobServer,
		heartbeatServices,
		metricRegistry,
		archivedExecutionGraphStore,
		new AkkaQueryServiceRetriever(
			metricQueryServiceActorSystem,
			Time.milliseconds(configuration.getLong(WebOptions.TIMEOUT))),this);
	
	```


2. createDispatcherResourceManagerComponentFactory由YarnSessionClusterEntrypoint重写生成SessionDispatcherResourceManagerComponentFactory实例，clusterComponent为SessionDispatcherResourceManagerComponent；clusterComponent的dispatcherFactory为SessionDispatcherFactory，restEndpointFactory实例为SessionRestEndpointFactory，resourceManagerFactory为YarnResourceManagerFactory；

3. 在create方法内调用各factory生成实例，dispatcher实例为`org.apache.flink.runtime.dispatcher.StandaloneDispatcher`，负责接收client提交的任务，`org.apache.flink.runtime.dispatcher.DispatcherRestEndpoint.initializeHandlers`响应`org.apache.flink.runtime.rest.handler.job.JobSubmitHandler.handleRequest`事件，收到任务后调用dispatcher.submitJob->persistAndRunJob->runJob；调用createJobManagerRunner方法创建JobMaster；
4. `org.apache.flink.yarn.YarnResourceManager.start`调用后负责向yarn申请资源。

![flink非detched模式](/images/flink_yarn.jpg)
<center><font size="2" color=gray>flink非detched模式</font></center>

### jobmanager端detched模式

1. 在am中的主类为`org.apache.flink.yarn.entrypoint.YarnJobClusterEntrypoint`，同样调用父类方法依次进入runClusterEntrypoint()->startCluster()-> runCluster()；
2. 此处dispatcherResourceManagerComponentFactory由 `YarnJobClusterEntrypoint. createDispatcherResourceManagerComponentFactory` 创建生成`JobDispatcherResourceManagerComponentFactory`，clusterComponent为JobDispatcherResourceManagerComponent，clusterComponent的dispatcherFactory为JobDispatcherFactory，restEndpointFactory为JobRestEndpointFactory，resourceManagerFactory为YarnResourceManagerFactory；

3. 在`AbstractDispatcherResourceManagerComponentFactory.create` 方法内调用各factory生成实例，dispatcher由JobDispatcherFactory创建 `org.apache.flink.runtime.dispatcher.MiniDispatcher`，MiniDispatcher需要传入jobGraph参数，在DETACHED模式下，已将JobGraph创建成临时文件job.graph一并上传到am中，由FileJobGraphRetriever还原回JobGraph；dispatcher的submittedJobGraphStore为SingleJobSubmittedJobGraphStore；

4. 在调用dispatcher的grantLeadership会通过recoverJobs获取SingleJobSubmittedJobGraphStore的任务，通过tryAcceptLeadershipAndRunJobs调用runJob执行任务；调用createJobManagerRunner方法创建JobMaster。
5. `org.apache.flink.yarn.YarnResourceManager.start` 调用后负责向yarn申请资源。


![flinkdetched模式](/images/flink_yarn_detched.jpg)
<center><font size="2" color=gray>flink detched模式</font></center>
### flink提交命令

```shell
/data0/workspace/flink-1.7.0/bin/flink run \
-m yarn-cluster  \
-ynm  test-flink \
-c comXXX.PipelineMain \
-yn 2 \
-ys 2 \
-ytm 4096 \
-yjm 2048 \
-yst \
-p 4 \
-yD env.java.opts.taskmanager="-Dmetrics.database=test-flink" \
-yD env.java.opts.jobmanager="-Dmetrics.database=test-flink" \
-yarnqueue back1 \
-yd \
pipeline-flink-kafka-0-8-1.0-SNAPSHOT.jar \
test-flink
```

# spark streaming任务执行方式

spark的思想是把所有都作为批处理，即数据有界。

![spark streaming任务执行方式](/images/spark_stream.jpg)
<center><font size="2" color=gray>spark streaming任务执行方式</font></center>

1. 在StremingContext启动后会启动JobScheduler，scheduler会启动JobGenerator；
2. 在JobGenerator的RecurringTimer会按照duration时长定期生成GenerateJobs事件；
3. generateJobs方法响应此事件，调用DStreamGraph的generateJobs方法；
4. 在DStreamGraph中保存着所有DStream的依赖关系，会调用outputStream的generateJob即自后向前寻找依赖；
5. 最后找到inputStreams并调用comppute方法；
6. DStream的compute内部生成RDD，封装成Job由JobScheduler提交。

![spark streaming任务生成过程](/images/spark_job.jpg)
<center><font size="2" color=gray>spark streaming任务生成过程</font></center>


# flink stream任务执行方式

flink是流式处理思想，把所有的数据都看做是无界的。


![flink stream任务执行方式](/images/flink_stream.jpg)
<center><font size="2" color=gray>flink stream任务执行方式</font></center>

1. 通过Stream API编写的用户代码 —> StreamGraph；
2. StreamGraph转换成JobGraph，JobGraph是一个Job的用户逻辑视图表示；
3. 将JobGraph转换成ExecutionGraph，ExecutionGraph是JobGraph的并行表示。

![](/images/job_and_execution_graph.jpeg)

参考：
* [Apache Flink：特性、概念、组件栈、架构及原理分析](http://shiyanjun.cn/archives/1508.html)
* [Flink之用户代码生成调度层图结构](https://www.jianshu.com/p/13070729289c)