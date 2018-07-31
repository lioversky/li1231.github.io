# Spark UDF and functions


## 1.创建与使用udf

* udf用法一：注册（可在sql中使用）

	- java:
	
		``` 
		import org.apache.spark.sql.api.java.UDF1;
		import org.apache.spark.sql.types.DataTypes;
		sparkSession.udf().register("split", new UDF1<String, String[]>() {
	      public String[] call(String s) throws Exception {
	        return s.split(",");
	      }
	    }, DataTypes.createArrayType(DataTypes.StringType));
	    //sparkSession.udf().register("split", (String value) -> value.split(","),DataTypes.createArrayType(DataTypes.StringType));
		```
		
	- scala:
		
		```
		spark.udf.register("split", (value: String) => {
	      value.split(",")
	    })
		```

* udf用法二：方法调用

	- java:
	
		spark >= 2.3
		
		```
		import static org.apache.spark.sql.functions.*;
		import org.apache.spark.sql.expressions.UserDefinedFunction;
		UserDefinedFunction mode = udf(
		  (Seq<String> ss) -> ss.headOption(), DataTypes.StringType
		);
		df.select(mode.apply(col("vs"))).show();
	
		```
		spark < 2.3
		
		```
		UDF1 mode = new UDF1<Seq<String>, String>() {
		public String call(final Seq<String> types) throws Exception {
			return types.headOption();
		  }
		};
		sparkSession.udf().register("mode", mode, DataTypes.StringType);
		df.select(callUDF("mode", col("vs"))).show();
		df.selectExpr("mode(vs)").show();
		
		```
		
	- scala:
	
		```
		import org.apache.spark.sql.functions._
		val test_split = udf((value:String)=> value.split(","))
		ds.withColumn("test_split",test_split($"column"))
		```

## 2.String* 参数传入数组

使用scala时，dataset的select和drop等方法中有要传入String*可变参数类型的，但如果只有数组形式，转换方法如下：
` dataset.drop(columns: _*)` 或 `dataset.select(columns.map(col(_)): _*)`


## 3.case when

* sql语句中使用

* 方法中调用

## 4.struct

The star (*) can also be used to include all columns in a nested struct.

```
// input
{
  "a": 1,
  "b": 2
}

 Scala: events.select(struct("*") as 'x)
 SQL: select struct(*) as x from events

// output
{
  "x": {
    "a": 1,
    "b": 2
  }
}
```

## 5.get\_json_object

```
import org.apache.spark.sql.functions._

var streamingSelectDF = 
  streamingInputDF
   .select(get_json_object(($"value").cast("string"), "$.zip").alias("zip"))
    .groupBy($"zip") 
    .count()
```

## 6.json_tuple

json_tuple() can be used to extract fields available in a string column with JSON data.

```
// input
{
  "a": "{\"b\":1}"
}

Python: events.select(json_tuple("a", "b").alias("c"))
Scala:  events.select(json_tuple('a, "b") as 'c)
SQL:    select json_tuple(a, "b") as c from events

// output
{ "c": 1 }
```



[Structured Streaming Kafka Integration](https://docs.databricks.com/_static/notebooks/structured-streaming-etl-kafka.html) 

[Working with Complex Data Formats with Structured Streaming in Apache Spark 2.1](https://databricks.com/blog/2017/02/23/working-complex-data-formats-structured-streaming-apache-spark-2-1.html)