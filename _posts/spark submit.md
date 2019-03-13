# spark submit提交过程

本篇重点讨论spark on yarn的提交过程，提交使用spark-submit命令。

## 命令生成

使用spark-submit提交命令，是重新调用了spark-class脚本

```shell
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```

在spark-class内部启动java进程org.apache.spark.launcher.Main生成最后的启动命令，更加详细文档可参考：[Apache Spark源码剖析-Shell](http://www.winseliu.com/blog/2016/05/08/rrc-apache-spark-source-inside-shell/)

## 提交任务

1. 脚本在本地启动主类`org.apache.spark.deploy.SparkSubmit`，根据参数生成childArgs, childClasspath, sparkConf, childMainClass等；
2. 如果是yarn client模式，反射调用用户通过--class自定义的主类方法，在生成SparkContext时通过`YarnClientSchedulerBackend.submitApplication`启动ApplicationMaster调用`registerAM`注册成为AM并向ResourceManager申请资源；
3. 如果是cluster模式，SparkSubmit调用`org.apache.spark.deploy.yarn.YarnClusterApplication`启动`org.apache.spark.deploy.yarn.Client.run`方法，在submitApplication方法内生成yarnClient提交命令、上传所需要jar包等动作，并不执行用户代码。申请到的container中启动org.apache.spark.deploy.yarn.ApplicationMaster后调用run方法，在run执行`runDriver`方法，在`runDriver`中首先以独立线程启动用户main方法，然后调用`registerAM`注册成为AM并向ResourceManager申请资源；
5. 申请到资源后，在Container中启动CoarseGrainedExecutorBackend，来启动Executor。






