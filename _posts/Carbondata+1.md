# Carbondata整理 一

今天试运行了一下离线spark数据写入Carbondata，Carbondata使用1.5.0并自行编译的jar包，spark使用2.3.2。maven中央库的提供Carbondata的jar包spark好像是2.2.X版本，另外Carbondata-1.5.0好像对spark2.3只支持spark2.3.2，2.3.1api有不兼容的地方（也可能是我没用对）。

## 集成部署

本地编译命令：`mvn -DskipTests -Pspark-2.3 -Dspark.version=2.3.2 clean package`，成功后jar包位置：assembly/target/scala-2.11/apache-carbondata-1.5.0-SNAPSHOT-bin-spark2.2.1-hadoop2.7.2.jar，将jar包直接复制到spark依赖包中。

## 路径问题
上来先是跑demo，使用本地模式按照官方的例子，有几个路径需要说明一下：

1. javax.jdo.option.ConnectionURL在本地测试时指定的本地metadata_db路径，为sparkSession参数，值为：`jdbc:derby:;databaseName={metaDB};create=true`。此处明确指定metadata所存储的位置；
2. metaStorePath也为指定本地metadata_db路径，为getOrCreateCarbonSession方法的第一个参数。在任何位置未指定javax.jdo.option.ConnectionURL属性时且metaStorePath不为空，则metaDB按当前目前和metaStorePath拼接，规则为：`currentPath + metaStorePath + "/metastore_db"`;
3. storePath 数据存储目录，为getOrCreateCarbonSession方法的第二个参数。在目录内按照库名/表名存储数据；
4. spark.sql.warehouse.dir是spark sql仓库目录，为sparkSession参数。在Carbondata里如果未指定storePath，则使用此位置存储数据。


## 本地测试创建CarbonSession

* java创建

	```
	 CarbonProperties.getInstance()
        .addProperty(CarbonCommonConstants.CARBON_TIMESTAMP_FORMAT, "yyyy/MM/dd HH:mm:ss")
        .addProperty(CarbonCommonConstants.CARBON_DATE_FORMAT, "yyyy/MM/dd")
        .addProperty(CarbonCommonConstants.ENABLE_UNSAFE_COLUMN_PAGE, "true")
        .addProperty(CarbonCommonConstants.CARBON_BADRECORDS_LOC, "");

    String path = (String) config.getApplicationConfig().getConfig().get("carbonPath");

    String storeDB = path + "meta/metastore_db";
    String storePath = "file://" + path + "store";
    String warehouse = "file://" + path + "warehouse";

    SparkSession sparkSession = new CarbonBuilder(CarbonSession
        .builder()
        .enableHiveSupport()
        .config("spark.sql.warehouse.dir", warehouse)
        .config("javax.jdo.option.ConnectionURL",
            String.format("jdbc:derby:;databaseName=%s;create=true", storeDB))
        .config("spark.driver.host", "localhost")
    )
        .getOrCreateCarbonSession(storePath);
	```

* scala 创建

	```
	val storeDB = path + "meta/metastore_db"
    val storePath = "file://" + path + "store"
    val warehouse = "file://" + path + "warehouse"

    import org.apache.spark.sql.SparkSession
    import org.apache.spark.sql.CarbonSession._
    val carbon = SparkSession.builder()
      .enableHiveSupport()
      .config(sc.getConf)
      .config("spark.sql.warehouse.dir", warehouse)
      .config("javax.jdo.option.ConnectionURL",
        String.format("jdbc:derby:;databaseName=%s;create=true", storeDB))
      .getOrCreateCarbonSession(storePath)

	```

## 集成spark测试

与spark集成时，增加hive-site.xml配置文件，将参数写到配置文件中即可，metadata存储在mysql中，数据保存在hdfs。配置文件如下：

```xml
<configuration>

  <!--<property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:derby:memory:databaseName=metastore_db;create=true</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.apache.derby.jdbc.EmbeddedDriver</value>
  </property>

  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>file://${user.dir}/hive/warehouse</value>
  </property>-->

  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>hdfs://nyx/user/hongxun/carbon/warehouse</value>
  </property>
  <property>
    <name>spark.sql.warehouse.dir</name>
    <value>hdfs://nyx/user/hongxun/carbon/warehouse</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://d136092.innet.dip.weibo.com:3307/carbondata?createDatabaseIfNotExist=true</value>
  </property>
      
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
  </property>
      
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
  </property>
      
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>mysqladmin</value>
  </property>

</configuration>
```

调用`dataset.write().format("carbondata").options(options).save()`即会按照dataset的schema保存，spark会自动创建库表并写入数据，options需要指定"tableName"。

当刚创建完表后，短时间内每次drop table时都会报错，大概是有锁，让稍后再试，然后隔一天后再试，表就可以删除，不知道是什么策略。


## partitions数

在执行大量的数据过程中，发现在写入carbondata的task数都固定在8个，无论有没有蹭shuffle。

在日志中看到如下内容：

> INFO DataLoadPartitionCoalescer: Driver partition: 1655, no locality: 1655 

> INFO DataLoadPartitionCoalescer: Driver no locality partition
  
> INFO SparkContext: Starting job: collect at CarbonDataRDDFactory.scala:1120 

>  INFO DAGScheduler: Got job 0 (collect at CarbonDataRDDFactory.scala:1120) with 8 output partitions 

可以看到，最后的partition并未使用原始的partition数，导致计算速度变慢。

从源码中找到，最后的action操作为：`CarbonDataRDDFactory.loadDataFrame()`的`NewDataFrameLoaderRDD.collect()`，而NewDataFrameLoaderRDD是按照如下生成实例：

```
  val rdd = dataFrame.get.rdd
  //主机节点个数
  val nodeNumOfData = rdd.partitions.flatMap[String, Array[String]] { p =>
    DataLoadPartitionCoalescer.getPreferredLocs(rdd, p).map(_.host)
  }.distinct.length
  val nodes = DistributionUtil.ensureExecutorsByNumberAndGetNodeList(
    nodeNumOfData,
    sqlContext.sparkContext)
  val newRdd = new DataLoadCoalescedRDD[Row](sqlContext.sparkSession, rdd, nodes.toArray
    .distinct)

  new NewDataFrameLoaderRDD(
    sqlContext.sparkSession,
    new DataLoadResultImpl(),
    carbonLoadModel,
    newRdd
  )
```

而下是`DataLoadCoalescedRDD`中的`internalGetPartitions`方法决定最后的partition数，而在此方法内正是调用`DataLoadPartitionCoalescer.run`才打印了上面的日志信息，在`DataLoadPartitionCoalescer.run`中存在变量`noLocality`来判断最后是否使用原始partition数，nodes数即为所运行task的独立机器数。

```
def run(): Array[Partition] = {
    // 1. group partitions by node
    groupByNode()
    LOGGER.info(s"partition: ${prevPartitions.length}, no locality: ${noLocalityPartitions.length}")
    val partitions = if (noLocality) {
      // 2.A no locality partition
      repartitionNoLocality()
    } else {
      // 2.B locality partition
      repartitionLocality()
    }
    DataLoadPartitionCoalescer.checkPartition(prevPartitions, partitions)
    partitions
  }
```

## 测试

### 简单表测试

测试写入的数据总条数为443786680，共12列，维度列10列。使用compress=true后大小为4G（不加此参数也一样）。以下测试是在单机上使用local[*]进行的测试。

* select count(*) from table 立即返回;
* select count(domain) from table where domain = 'XXXX' 1秒返回；
* select count(distinct domain) from table 16秒；
* 对所有维度列做group by，对指标列求sum和count 1分；

### 分区表测试

