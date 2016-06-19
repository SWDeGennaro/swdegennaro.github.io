---
layout: post
title: Running Scala scripts in Spark shell
---

One of the great features of Apache Spark is the Spark Shell. It allows for interactive analysis of data from a variaty of data sources, be that HDFS, S3 or just your local file system. The spark shell allows you to quickly prototype your ETL jobs using all or a subset of your
data without the need to build package and keep deploying your application to the cluster.

There is a problem however as your simple prototype grows you find yourself copying and pasting from your favourite text editor and into the shell, or vice versa. Something I found out recently was that you can run the shell by passing in a Scala file with all of your commands that you would pass into the shell. 

This is a huge time saver and I was surprised I hadn't come across this feature during the year or so I've been using Spark. Don't get me wrong I've learnt other shortcuts with the shell like building an application jar and then loading it into the shell using `--jars some-application.jar` which then allows you to do `import com.somecompany.someapplication.Functionality` and use the functionality you have built up in your application in order to test and debug.

## Simple Example

Below is a simple example of what one of these Scala scripts for want of a better word might look like.

```
//load data from HDFS parquet file
val clickStreamDf = sqlContext.read.parquet("hdfs:///raw/2016/05/26/clickstream.parquet").repartition(100).cache()

/*
* schema 
* url | timestamp | domain
*/
clickStreamDf.registerTempTable("click_stream")

//force execution causing repartition and caching of the data from HDFS
clickStreamDf.count()

val countsByDomainDf =
	sqlContext.sql("""
	| SELECT COUNT(*)
	| FROM click_stream as cs
	| GROUP BY cs.domain
	""".stripMargin)

// output results to HDFS
countsByDomainDf.rdd.saveAsTextFile("hdfs:///output/2016/05/26")

```
And from the shell assuming the file is on the master node of your cluster (or your local machine if doing this locally). 

```
$SPARK_HOME/bin/spark-shell -i ClickStreamEtl.scala
```
