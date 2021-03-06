### Introduction

This tutorial walks you through many of the newer features of Spark 1.6 on YARN.

With YARN, Hadoop can now support many types of data and application workloads; Spark on YARN becomes yet another workload running against the same set of hardware resources.

The tutorial describes how to:

* Install Spark 1.6 Tech Preview on HDP 2.3.x
* Run Spark on YARN and run the canonical Spark examples: SparkPi and Wordcount.
* Run Spark 1.6 on HDP 2.4.
* Use the Spark DataFrame API.
* Read/write data from Hive.
* Use SparkSQL Thrift Server for JDBC/ODBC access.
* Use ORC files with Spark, with examples.
* Use SparkR.
* Use the DataSet API.

When you are ready to go beyond these tasks, checkout the [Apache Spark Machine Learning Library (MLlib) Guide](http://spark.apache.org/docs/latest/mllib-guide.html).

### Prerequisites

This tutorial is a part of series of hands-on tutorials to get you started with HDP using Hortonworks sandbox. Please ensure you complete the prerequisites before proceeding with this tutorial.

*   Downloaded and Installed latest [Hortonworks Sandbox](http://hortonworks.com/products/hortonworks-sandbox/#install)
*   [Learning the Ropes of the Hortonworks Sandbox](http://hortonworks.com/hadoop-tutorial/learning-the-ropes-of-the-hortonworks-sandbox/)

#### Installing Spark 1.6 on HDP 2.3.x

If you are running HDP 2.3.x you can install Spark 1.6 Technical Preview (TP).

The Spark 1.6 TP is provided in RPM and DEB package formats. The following instructions assume RPM packaging:

##### 1. Download the Spark 1.6 RPM repository:

~~~ bash
wget -nv http://private-repo-1.hortonworks.com/HDP/centos6/2.x/updates/2.3.4.1-10/hdp.repo -O /etc/yum.repos.d/HDP-TP.repo
~~~

For installing on Ubuntu use the following

~~~ bash
http://private-repo-1.hortonworks.com/HDP/ubuntu12/2.x/updates/2.3.4.1-10/hdp.list
~~~

##### 2. Install the Spark Package:

Download the Spark 1.6 RPM (and pySpark, if desired) and set it up on your HDP 2.3 cluster:

~~~ bash
yum install spark_2_3_4_1_10-master -y
~~~

If you want to use pySpark, install it as follows and make sure that Python is installed on all nodes.

~~~ bash
yum install spark_2_3_4_1_10-python -y
~~~

The RPM installer will also download core Hadoop dependencies. It will create “spark” as an OS user, and it will create the `/user/spark directory` in HDFS.

##### 3. Set JAVA_HOME:

Make sure that you set JAVA_HOME before you launch the Spark Shell or thrift server.

~~~ bash
export JAVA_HOME=<path to JDK 1.8>
~~~

##### 4. Create hive-site in the Spark conf directory:

As user root, create the file `SPARK_HOME/conf/hive-site.xml`. Edit the file to contain only the following configuration setting:

~~~ xml
<configuration>
  <property>
    <name>hive.metastore.uris</name>
    <!--Make sure that <value> points to the Hive Metastore URI in your cluster -->
    <value>thrift://sandbox.hortonworks.com:9083</value>
    <description>URI for client to contact metastore server</description>
  </property>
</configuration>
~~~

##### 5. Set SPARK_HOME

If you haven't already, make sure to set `SPARK_HOME` before proceeding:

~~~ bash
export SPARK_HOME=/usr/hdp/current/spark-client
~~~

#### Run the Spark Pi Example

To test compute intensive tasks in Spark, the Pi example calculates pi by “throwing darts” at a circle. The example points in the unit square ((0,0) to (1,1)) and sees how many fall in the unit circle. The fraction should be pi/4, which is used to estimate Pi.

To calculate Pi with Spark:

**Change to your Spark directory and become spark OS user:**

~~~ bash
cd $SPARK_HOME
su spark
~~~

**Run the Spark Pi example in yarn-client mode:**

~~~ bash
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-client --num-executors 3 --driver-memory 512m --executor-memory 512m --executor-cores 1 lib/spark-examples*.jar 10
~~~

**Note:** The Pi job should complete without any failure messages and produce output similar to below, note the value of Pi in the output message:

~~~ js
...
16/02/25 21:27:11 INFO YarnScheduler: Removed TaskSet 0.0, whose tasks have all completed, from pool
16/02/25 21:27:11 INFO DAGScheduler: Job 0 finished: reduce at SparkPi.scala:36, took 19.346544 s
Pi is roughly 3.143648
16/02/25 21:27:12 INFO ContextHandler: stopped o.s.j.s.ServletContextHandler{/metrics/json,null}
...
~~~

### Using WordCount with Spark

#### Copy input file for Spark WordCount Example

Upload the input file you want to use in WordCount to HDFS. You can use any text file as input. In the following example, log4j.properties is used as an example:

As user `spark`:

~~~ bash
hadoop fs -copyFromLocal /etc/hadoop/conf/log4j.properties /tmp/data
~~~

### Run Spark WordCount

To run WordCount:

#### Run the Spark shell:

~~~ bash
./bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m
~~~

Output similar to the following will be displayed, followed by a `scala>` REPL prompt:

~~~ bash
Welcome to
     ____              __
    / __/__  ___ _____/ /__
   _\ \/ _ \/ _ `/ __/  '_/
  /___/ .__/\_,_/_/ /_/\_\   version 1.6.0
     /_/
Using Scala version 2.10.5 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_60)
Type in expressions to have them evaluated.
Type :help for more information.
15/12/16 13:21:57 INFO SparkContext: Running Spark version 1.6.0
...

scala>
~~~

At the `scala` REPL prompt enter:

~~~ js
val file = sc.textFile("/tmp/data")
val counts = file.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
~~~

Save `counts` to a file:

~~~ js
counts.saveAsTextFile("/tmp/wordcount")
~~~

##### Viewing the WordCount output with Scala Shell

To view the output, at the `scala>` prompt type:

~~~ js
counts.count()
~~~

You should see an output screen similar to:

~~~ js
...
16/02/25 23:12:20 INFO DAGScheduler: Job 1 finished: count at <console>:32, took 0.541229 s
res1: Long = 341

scala>
~~~

To print the full output of the WordCount job type:

~~~ js
counts.toArray().foreach(println)
~~~

You should see an output screen similar to:

~~~ js
...
((Hadoop,1)
(compliance,1)
(log4j.appender.RFAS.layout.ConversionPattern=%d{ISO8601},1)
(additional,1)
(default,2)

scala>
~~~

##### Viewing the WordCount output with HDFS

To read the output of WordCount using HDFS command:  
Exit the Scala shell.

~~~ bash
exit
~~~

View WordCount Results:

~~~ bash
hadoop fs -ls /tmp/wordcount
~~~

You should see an output similar to:

~~~ bash
/tmp/wordcount/_SUCCESS
/tmp/wordcount/part-00000
/tmp/wordcount/part-00001
~~~

Use the HDFS `cat` command to see the WordCount output. For example,

~~~ bash
hadoop fs -cat /tmp/wordcount/part-00000
~~~

##### Using the Spark DataFrame API

 DataFrame API provides easier access to data since it looks conceptually like a Table and a lot of developers from Python/R/Pandas are familiar with it.

As a `spark` user, upload people.txt and people.json files to HDFS:

~~~ bash
cd /usr/hdp/current/spark-client
su spark
hadoop fs -copyFromLocal examples/src/main/resources/people.txt people.txt
hadoop fs -copyFromLocal examples/src/main/resources/people.json people.json
~~~

As a `spark` user, launch the Spark Shell:

~~~ bash
cd /usr/hdp/current/spark-client
su spark
./bin/spark-shell --num-executors 2 --executor-memory 512m --master yarn-client
~~~

At a `scala>` REPL prompt, type the following:

~~~ js
val df = sqlContext.jsonFile("people.json")
~~~

Using `df.show`, display the contents of the DataFrame:

~~~ js
df.show
~~~

You should see an output similar to:

~~~ js
...
15/12/16 13:28:15 INFO YarnScheduler: Removed TaskSet 2.0, whose tasks have all completed, from pool
+----+-------+
| age|   name|
+----+-------+
|null|Michael|
|  30|   Andy|
|  19| Justin|
+----+-------+

scala>
~~~

##### Additional DataFrame API examples

Continuing at the `scala>` prompt, type the following:

~~~ js
import org.apache.spark.sql.functions._ 
~~~

~~~ js
// Select all, but increment the age by 1
df.select(df("name"), df("age") + 1).show()
~~~

This will produce an output similar to the following:

~~~ bash
...
+-------+---------+
|   name|(age + 1)|
+-------+---------+
|Michael|     null|
|   Andy|       31|
| Justin|       20|
+-------+---------+

scala>
~~~

~~~ js
// Select people older than 21
df.filter(df("age") > 21).show()
~~~

This will produce an output similar to the following:

~~~ bash
...
+---+----+
|age|name|
+---+----+
| 30|Andy|
+---+----+

scala>
~~~


~~~ js
// Count people by age
df.groupBy("age").count().show()
~~~

This will produce an output similar to the following:

~~~ bash
...
+----+-----+
| age|count|
+----+-----+
|null|    1|
|  19|    1|
|  30|    1|
+----+-----+

scala>
~~~

##### Programmatically Specifying Schema

~~~ js
import org.apache.spark.sql._ 

// Create sql context from an existing SparkContext (sc)
val sqlContext = new org.apache.spark.sql.SQLContext(sc)  

// Create people RDD
val people = sc.textFile("people.txt")                    

// Encode schema in a string
val schemaString = "name age"

// Import Spark SQL data types and Row
import org.apache.spark.sql.types.{StructType,StructField,StringType} 

// Generate the schema based on the string of schema
val schema = 
  StructType(
    schemaString.split(" ").map(fieldName => StructField(fieldName, StringType, true)))

// Convert records of people RDD to Rows
val rowRDD = people.map(_.split(",")).map(p => Row(p(0), p(1).trim))

// Apply the schema to the RDD
val peopleSchemaRDD = sqlContext.createDataFrame(rowRDD, schema)

// Register the SchemaRDD as a table
peopleSchemaRDD.registerTempTable("people")

// Execute a SQL statement on the 'people' table
val results = sqlContext.sql("SELECT name FROM people")

// The results of SQL queries are SchemaRDDs and support all the normal RDD operations.
// The columns of a row in the result can be accessed by ordinal
results.map(t => "Name: " + t(0)).collect().foreach(println)
~~~

This will produce an output similar to the following:

~~~ js
15/12/16 13:29:19 INFO DAGScheduler: Job 9 finished: collect at :39, took 0.251161 s
15/12/16 13:29:19 INFO YarnHistoryService: About to POST entity application_1450213405513_0012 with 10 events to timeline service http://green3:8188/ws/v1/timeline/
Name: Michael
Name: Andy
Name: Justin

scala>
~~~

### Running Hive UDF

Hive has a built-in UDF collection `collect_list(col)`, which returns a list of objects with duplicates.  

The example below reads and writes to HDFS under Hive directories.

Before running Hive examples run the following steps:

#### Launch Spark Shell on YARN cluster

~~~ bash
su hdfs

# If not already in spark-client directory, change to that directory
cd $SPARK_HOME
./bin/spark-shell --num-executors 2 --executor-memory 512m --master yarn-client
~~~

#### Create Hive Context

At a `scala>` REPL prompt type the following:

~~~ js
val hiveContext = new org.apache.spark.sql.hive.HiveContext(sc)
~~~

You should see an output similar to the following:

~~~ bash
...
16/02/25 20:13:33 INFO SessionState: Created local directory: /tmp/root/f3b7d0fb-031f-4e69-a792-6f12ed7ffa03
16/02/25 20:13:33 INFO SessionState: Created HDFS directory: /tmp/hive/root/f3b7d0fb-031f-4e69-a792-6f12ed7ffa03/_tmp_space.db
hiveContext: org.apache.spark.sql.hive.HiveContext = org.apache.spark.sql.hive.HiveContext@30108473

scala>
~~~

#### Create Hive Table

~~~ js
hiveContext.sql("CREATE TABLE IF NOT EXISTS TestTable (key INT, value STRING)")
~~~

You should see an output similar to the following:

~~~ js
...
16/02/25 20:15:10 INFO PerfLogger: </PERFLOG method=releaseLocks start=1456431310152 end=1456431310153 duration=1 from=org.apache.hadoop.hive.ql.Driver>
16/02/25 20:15:10 INFO PerfLogger: </PERFLOG method=Driver.run start=1456431295549 end=1456431310153 duration=14604 from=org.apache.hadoop.hive.ql.Driver>
res0: org.apache.spark.sql.DataFrame = [result: string]

scala>
~~~

#### Load example KV value data into Table

~~~ js
hiveContext.sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE TestTable")
~~~

You should see an output similar to the following:

~~~ js
16/02/25 20:19:17 INFO PerfLogger: </PERFLOG method=releaseLocks start=1456431557360 end=1456431557360 duration=0 from=org.apache.hadoop.hive.ql.Driver>
16/02/25 20:19:17 INFO PerfLogger: </PERFLOG method=Driver.run start=1456431555555 end=1456431557360 duration=1805 from=org.apache.hadoop.hive.ql.Driver>
res1: org.apache.spark.sql.DataFrame = [result: string]

scala>
~~~

#### Invoke Hive collect_list UDF

~~~ js
hiveContext.sql("from TestTable SELECT key, collect_list(value) group by key order by key").collect.foreach(println)
~~~

You should see output similar to the following:

~~~ js
...
[496,WrappedArray(val_496)]
[497,WrappedArray(val_497)]
[498,WrappedArray(val_498, val_498, val_498)]

scala>
~~~

### Reading & Writing ORC Files

Hortonworks worked in the community to bring full ORC support to Spark. Recently we blogged about using [ORC with Spark](http://hortonworks.com/blog/bringing-orc-support-into-apache-spark/). See the blog post for all ORC examples, with advances such as partition pruning and predicate pushdown.

### Using SparkSQL Thrift Server for JDBC/ODBC Access

SparkSQL’s thrift server provides JDBC access to SparkSQL.


**Create logs directory**

Change ownership of `logs` directory from `root` to `spark` user:

~~~ bash
cd $SPARK_HOME
chown spark:hadoop logs
~~~

**Start Thrift Server**  


~~~ bash
su spark
./sbin/start-thriftserver.sh --master yarn-client --executor-memory 512m --hiveconf hive.server2.thrift.port=10015
~~~

**Connect to the Thrift Server over Beeline**

Launch Beeline:

~~~ bash
./bin/beeline
~~~

**Connect to Thrift Server and issue SQL commands**  

On the `beeline>` prompt type:

~~~ sql
!connect jdbc:hive2://localhost:10015
~~~

***Notes***
* This example does not have security enabled, so any username-password combination should work.
* The connection may take a few seconds to be available in a Sandbox environment.

You should see an output similar to the following:

~~~
beeline> !connect jdbc:hive2://localhost:10015
Connecting to jdbc:hive2://localhost:10015
Enter username for jdbc:hive2://localhost:10015:
Enter password for jdbc:hive2://localhost:10015:
...
Connected to: Spark SQL (version 1.6.0)
Driver: Spark Project Core (version 1.6.0.2.4.0.0-169)
Transaction isolation: TRANSACTION_REPEATABLE_READ
0: jdbc:hive2://localhost:10015>
~~~

Once connected, try `show tables`:

~~~ sql
show tables;
~~~

You should see an output similar to the following:

~~~ bash
+------------+--------------+--+
| tableName  | isTemporary  |
+------------+--------------+--+
| sample_07  | false        |
| sample_08  | false        |
| testtable  | false        |
+------------+--------------+--+
3 rows selected (2.399 seconds)
0: jdbc:hive2://localhost:10015>
~~~

Type `Ctrl+C` to exit beeline.

*   **Stop Thrift Server**

~~~ bash
./sbin/stop-thriftserver.sh
~~~

### Running SparkR

**Prerequisites**

Before you can run SparkR, you need to install R linux distribution by following these steps as a `root` user:

~~~ bash
cd $SPARK_HOME
su -c 'rpm -Uvh http://download.fedoraproject.org/pub/epel/5/i386/epel-release-5-4.noarch.rpm'
sudo yum update
sudo yum install R
~~~

**Launch SparkR**

~~~ bash
su spark
cd $SPARK_HOME
./bin/sparkR
~~~

You should see an output similar to the following:

~~~ js
...
Welcome to
   ____              __
  / __/__  ___ _____/ /__
 _\ \/ _ \/ _ `/ __/  '_/
/___/ .__/\_,_/_/ /_/\_\   version  1.6.0
   /_/


Spark context is available as sc, SQL context is available as sqlContext
>
~~~

Create a DataFrame and list the first few lines:

~~~ js
sqlContext <- sparkRSQL.init(sc)
~~~

~~~ js
df <- createDataFrame(sqlContext, faithful)
~~~

~~~ js
head(df)
~~~

You should see an output similar to the following:

~~~ js
...
eruptions waiting
1     3.600      79
2     1.800      54
3     3.333      74
4     2.283      62
5     4.533      85
6     2.883      55
>
~~~

Create people DataFrame from 'people.json' and list the first few lines:

~~~ js
people <- read.df(sqlContext, "people.json", "json")
~~~

~~~ js
head(people)
~~~

You should see an output similar to the following:

~~~ js
...
age    name
1  NA Michael
2  30    Andy
3  19  Justin
~~~

For additional SparkR examples, see https://spark.apache.org/docs/latest/sparkr.html.

To exit SparkR type:

~~~ js
quit()
~~~

### Using the DataSet API

The Spark Dataset API brings the best of RDD and Data Frames together, for type safety and user functions that run directly on existing JVM types.

**Launch Spark**

As `spark` user, launch the Spark Shell:

~~~ bash
cd $SPARK_HOME
su spark
./bin/spark-shell --num-executors 2 --executor-memory 512m --master yarn-client
~~~

At the `scala>` prompt, copy & paste the following:

~~~ js
val ds = Seq(1, 2, 3).toDS()
ds.map(_ + 1).collect() // Returns: Array(2, 3, 4)

// Encoders are also created for case classes.
case class Person(name: String, age: Long)
val ds = Seq(Person("Andy", 32)).toDS()

// DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name.
val path = "people.json"
val people = sqlContext.read.json(path).as[Person]
~~~
To view contents of people type:

~~~ js
people.show()
~~~

You should see an output similar to the following:

~~~
...
+----+-------+
| age|   name|
+----+-------+
|null|Michael|
|  30|   Andy|
|  19| Justin|
+----+-------+

scala>
~~~

To exit type `exit`.

### Running the Machine Learning Spark Application

To optimize MLlib performance, install the [netlib-java](https://github.com/fommil/netlib-java) native library. If this native library is not available at runtime, you will see a warning message and a pure JVM implementation will be used instead.

To use MLlib in Python, you will need [NumPy](http://www.numpy.org/) version 1.4 or later.

For Spark ML examples, visit: http://spark.apache.org/docs/latest/mllib-guide.html.

### Additional Resources
For more tutorials on Spark, visit: [http://hortonworks.com/tutorials](http://hortonworks.com/tutorials).

If you have questions about Spark and Data Science, checkout the [Hortonworks Community Connection](https://community.hortonworks.com/spaces/85/data-science.html?type=question).
