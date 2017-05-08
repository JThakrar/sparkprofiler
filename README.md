# SparkProfiler Overview
This project shows how "events" generated by Spark applications can be analyzed to profile those applications.
Profiling here means understanding how and where an application spent its time, the amount of processing it did, its memory footprint, etc.
Since Apache Spark is a distributed processing framework, this kind of processing helps understand application resource utilization and provides a framework to optimize and tune applications.

Apache Spark uses events internally to coordinate its activities. By setting the following configuration parameters, the events can be logged to a file as JSON entries
- spark.eventLog.enabled = true
- spark.eventLog.dir = dir-path

Note that the events are logged at the driver level, so for local (non-clustered) Spark applications and for YARN-client mode applications, <dir-path> can be a local directory. And for YARN cluster applications, <dir-path> is an HDFS location. The events are created underneath that base directory. This project has an analyzer that can process the events file and generate a summary for the application. 

# How to Build and Run
Clone this project and build the project using `sbt assembly`. This will generate an assembly jar file `target/scala-2.11/sparkprofiler-assembly-1.0-SNAPSHOT.jar`.
 
Identify a location for an application events file and generate a summary for it as follows:

```
    spark-submit --class SummaryGenerator target/scala-2.11/sparkprofiler-assembly-1.0-SNAPSHOT.jar <events-file>
```

SummaryGenerator above is a Spark application that is run locally. For (very) large applications, it is not uncommon to have an events file that is a few GB large and if needed, SummaryGenerator can also be run as a distributed cluster application.

# Interactive Analysis
The project contains 3 main top-level Scala objects:
- Parser : This contains methods to parse the Spark events text lines into appropriate kind of events
- Profiler : This contains methods to generate a summary or profile of Spark jobs, stages and tasks
- SummaryGenerator: This is a sample program that shows how to process and analyze events file.

Note that you can also carry out interactive analysis of events using spark-shell as follows:

```
spark-shell --jars target/scala-2.11/sparkprofiler-assembly-1.0-SNAPSHOT.jar
```

```
scala> import Parser._
import Parser._

scala> import Profiler._
import Profiler._

scala> val events = spark.read.textFile("../application_1479761269966_255535_1")
events: org.apache.spark.sql.Dataset[String] = [value: string]

scala> val application = parseApplication(spark, events)
application: Application = Application(CPM,application_1479761269966_255535,1483011710370,1483024267784,12557414)

scala> val jobs = parseJobs(spark, events, application)
jobs: org.apache.spark.sql.Dataset[Job] = [applicationId: string, applicationName: string ... 9 more fields]

scala> val stages = parseStages(spark, events, jobs)
stages: org.apache.spark.sql.Dataset[Stage] = [applicationId: string, applicationName: string ... 16 more fields]

scala> val tasks = parseTasks(spark, events, stages)
tasks: org.apache.spark.sql.Dataset[Task] = [applicationId: string, applicationName: string ... 45 more fields]

scala> tasks.printSchema
root
 |-- applicationId: string (nullable = true)
 |-- applicationName: string (nullable = true)
 |-- jobId: integer (nullable = true)
 |-- stageId: integer (nullable = true)
 |-- stageName: string (nullable = true)
 |-- stageAttemptId: integer (nullable = true)
 |-- taskId: integer (nullable = true)
 |-- taskAttemptId: integer (nullable = true)
 |-- taskType: string (nullable = true)
 |-- executorId: string (nullable = true)
 |-- host: string (nullable = true)
 |-- peakMemory: long (nullable = true)
 |-- inputRows: long (nullable = true)
 |-- outputRows: long (nullable = true)
 |-- locality: string (nullable = true)
 |-- speculative: boolean (nullable = true)
 |-- applicationStartTime: long (nullable = true)
 |-- applicationEndTime: long (nullable = true)
 |-- jobStartTime: long (nullable = true)
 |-- jobEndTime: long (nullable = true)
 |-- stageStartTime: long (nullable = true)
 |-- stageEndTime: long (nullable = true)
 |-- taskStartTime: long (nullable = true)
 |-- taskEndTime: long (nullable = true)
 |-- failed: boolean (nullable = true)
 |-- taskDuration: long (nullable = true)
 |-- stageDuration: long (nullable = true)
 |-- jobDuration: long (nullable = true)
 |-- applicationDuration: long (nullable = true)
 |-- gcTime: long (nullable = true)
 |-- gettingResultTime: long (nullable = true)
 |-- resultSerializationTime: long (nullable = true)
 |-- resultSize: long (nullable = true)
 |-- dataReadMethod: string (nullable = true)
 |-- bytesRead: long (nullable = true)
 |-- recordsRead: long (nullable = true)
 |-- memoryBytesSpilled: long (nullable = true)
 |-- diskBytesSpilled: long (nullable = true)
 |-- shuffleBytesWritten: long (nullable = true)
 |-- shuffleRecordsWritten: long (nullable = true)
 |-- shuffleWriteTime: long (nullable = true)
 |-- remoteBlocksFetched: long (nullable = true)
 |-- localBlocksFetched: long (nullable = true)
 |-- fetchWaitTime: long (nullable = true)
 |-- remoteBytesRead: long (nullable = true)
 |-- localBytesRead: long (nullable = true)
 |-- totalRecordsRead: long (nullable = true)


scala> val taskInfo = taskProfile(spark, tasks)
taskInfo: org.apache.spark.sql.DataFrame = [summary: string, taskDuration: string ... 20 more fields]


scala> taskInfo.printSchema
root
 |-- summary: string (nullable = true)
 |-- taskDuration: string (nullable = true)
 |-- gcTime: string (nullable = true)
 |-- peakMemory: string (nullable = true)
 |-- gettingResultTime: string (nullable = true)
 |-- resultSerializationTime: string (nullable = true)
 |-- inputRows: string (nullable = true)
 |-- outputRows: string (nullable = true)
 |-- resultSize: string (nullable = true)
 |-- bytesRead: string (nullable = true)
 |-- recordsRead: string (nullable = true)
 |-- memoryBytesSpilled: string (nullable = true)
 |-- diskBytesSpilled: string (nullable = true)
 |-- shuffleBytesWritten: string (nullable = true)
 |-- shuffleRecordsWritten: string (nullable = true)
 |-- shuffleWriteTime: string (nullable = true)
 |-- remoteBlocksFetched: string (nullable = true)
 |-- localBlocksFetched: string (nullable = true)
 |-- fetchWaitTime: string (nullable = true)
 |-- remoteBytesRead: string (nullable = true)
 |-- localBytesRead: string (nullable = true)
 |-- totalRecordsRead: string (nullable = true)


scala> taskInfo.select("summary", "taskDuration", "peakMemory", "inputRows", "outputRows").show
+-------+-----------------+-------------------+--------------------+--------------------+
|summary|     taskDuration|         peakMemory|           inputRows|          outputRows|
+-------+-----------------+-------------------+--------------------+--------------------+
|  count|          3602572|            3602572|             3602572|             3602572|
|   mean| 1523.08107929557|4.361762662780258E7|6.486111500415648E10|2.6493850101408936E7|
| stddev|6057.808744537105|1.464018179287083E9|5.200738235692943E10| 4.771569890514521E7|
|    min|                3|                  0|                   0|                   0|
|    max|           164986|        96677396480|        156703387469|           133978499|
+-------+-----------------+-------------------+--------------------+--------------------+


scala> taskInfo.select("summary", "taskDuration", "shuffleBytesWritten", "shuffleRecordsWritten", "shuffleWriteTime").show
+-------+-----------------+-------------------+---------------------+--------------------+
|summary|     taskDuration|shuffleBytesWritten|shuffleRecordsWritten|    shuffleWriteTime|
+-------+-----------------+-------------------+---------------------+--------------------+
|  count|          3602572|            3602572|              3602572|             3602572|
|   mean| 1523.08107929557|  4189.475602430708|   173.10059257663693| 8.639802211187173E7|
| stddev|6057.808744537105| 14717.052579771967|    757.3031832715615|3.7762916800988054E8|
|    min|                3|                  0|                    0|                   0|
|    max|           164986|             235734|                12574|         17734656525|
+-------+-----------------+-------------------+---------------------+--------------------+

```
