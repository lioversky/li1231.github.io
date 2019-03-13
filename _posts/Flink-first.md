# Flink整理

## Flink on yarn 任务提交过程

### client端 

1. 在使用bin/flink脚本提交任务时，调用`org.apache.flink.client.cli.CliFrontend`类，在main方法中首先
然后获取全局配置信息：`GlobalConfiguration.loadConfiguration(configurationDirectory)`再生成命令行解析器；然后根据参数执行具体方法，本次只关注run方法；

2. 在run方法中先解析参数，然后创建PackagedProgram，进入`runProgram(customCommandLine, commandLine, runOptions, program)`；先创建ClusterDescriptor
`ClusterDescriptor<T> clusterDescriptor = customCommandLine.createClusterDescriptor(commandLine);`
生成子类实例org.apache.flink.yarn.YarnClusterDescriptor；在此处会根据是否传入-yd 判断Detach进入不同的代码段；

3. 如果为detach模式，先生成jobGraph，过程中调用`org.apache.flink.client.program.PackagedProgramUtils.createJobGraph`时，在此处分为两种方法执行：program entryPoint和interactive mode;interactive mode，在interactive模式下`org.apache.flink.client.program.OptimizerPlanEnvironment.getOptimizedPlan(packagedProgram)`中，通过`org.apache.flink.client.program.PackagedProgram.invokeInteractiveModeForExecution()`调用了自定义主类的main方法生成FlinkPlan；再调用`clusterDescriptor.deployJobCluster(clusterSpecification,jobGraph,runOptions.getDetachedMode())`提交jobGraph生成yarn job,最后关闭ClusterClient；

4. 如果为非detach模式，clusterDescriptor.deploySessionCluster(clusterSpecification)中生成am并启动YarnSessionClusterEntrypoint，完成后返回RestClusterClient实例；然后进入`executeProgram`方法，执行`org.apache.flink.client.program.restRestClusterClient的.run`方法，在此处分为两种方法执行：program entryPoint和interactive mode;interactive mode模式是通过`prog.invokeInteractiveModeForExecution()`反射自定义主类的`main`方法执行，由`org.apache.flink.streaming.api.environment.StreamContextEnvironment.execute`调用`ClusterClient.run`方法最后执行`org.apache.flink.yarn.RestClusterClient.submitJob(JobGraph)`提交任务。

### jobmanager端非detched模式

1. 在am中的主类为YarnSessionClusterEntrypoint，在main方法中通过`ClusterEntrypoint.runClusterEntrypoint(yarnSessionClusterEntrypoint)`调用到`startCluster`方法，再进入到`runCluster`，在此方法内首先由`initializeServices`创建所有本地服务实例,然后创建DispatcherResourceManagerComponent实例，代码如下：

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

### jobmanager端detched模式

1. 在am中的主类为`org.apache.flink.yarn.entrypoint.YarnJobClusterEntrypoint`，同样调用父类方法依次进入runClusterEntrypoint()->startCluster()-> runCluster()；
2. 此处dispatcherResourceManagerComponentFactory由`YarnJobClusterEntrypoint. createDispatcherResourceManagerComponentFactory` 创建生成`JobDispatcherResourceManagerComponentFactory`，clusterComponent为JobDispatcherResourceManagerComponent，clusterComponent的dispatcherFactory为JobDispatcherFactory，restEndpointFactory为JobRestEndpointFactory，resourceManagerFactory为YarnResourceManagerFactory；

3. 在`AbstractDispatcherResourceManagerComponentFactory.create`方法内调用各factory生成实例，dispatcher由JobDispatcherFactory创建`org.apache.flink.runtime.dispatcher.MiniDispatcher`，MiniDispatcher需要传入jobGraph参数，在DETACHED模式下，已将JobGraph创建成临时文件job.graph一并上传到am中，由FileJobGraphRetriever还原回JobGraph；dispatcher的submittedJobGraphStore为SingleJobSubmittedJobGraphStore；

4. 在调用dispatcher的grantLeadership会通过recoverJobs获取SingleJobSubmittedJobGraphStore的任务，通过tryAcceptLeadershipAndRunJobs调用runJob执行任务；调用createJobManagerRunner方法创建JobMaster。
5. `org.apache.flink.yarn.YarnResourceManager.start`调用后负责向yarn申请资源。


## Flink任务生成过程

* 在每个算子生成过程中如addSource、addSink、flatMap等，都会调用DataStream的transform方法，最后调用`getExecutionEnvironment().addOperator(resultTransform)`将StreamTransformation加到StreamExecutionEnvironment的transformations中。
* 在env的execute方法中，通过`StreamGraph streamGraph = getStreamGraph()`生成StreamGraph实例，内部方法`StreamGraphGenerator.generate(this, transformations)`，StreamingJobGraphGenerator生成JobGraph；
* JobGraph中将所有的transformations分成一个或多个JobVertex，每个包含几个JobEdge
StreamingJobGraphGenerator

SlotManager

TaskManagerRunner->TaskManagerServices

org.apache.flink.runtime.jobgraph

TaskExecutor

StreamPartitioner



1. jobMaster在初始化的时候即调用createAndRestoreExecutionGraph方法创建executionGraph，而后JobManagerRunner.verifyJobSchedulingStatusAndStartJobManager中调用jobMaster.start方法，在start方法内依次startJobExecution()->resetAndScheduleExecutionGraph()，如果executionGraph.getState()不为CREATED，接着调用createAndRestoreExecutionGraph()->createExecutionGraph()，最后调用到ExecutionGraphBuilder.buildGraph；

2. 然后调用JobMaster的scheduleExecutionGraph分配执行；

Dispatcher.grantLeadership

