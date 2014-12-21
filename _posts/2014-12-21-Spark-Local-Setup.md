---
layout: post
title: Local Spark Setup
---

In this post I will explain the steps that I took in order to setup a development environment on my macbook, in order to run Spark jobs using HDFS and YARN. 

#### Install Hadoop & Spark

```
brew install hadoop apache-spark scala sbt
```

#### Configure Envrionment

```
vim ~/.bash_profile
```

Add the following environment variables to your bash profile 

```
export JAVA_HOME="$(/usr/libexec/java_home)"
export HADOOP_HOME="/usr/local/Cellar/hadoop/2.6.0"
export HADOOP_CONF_DIR="$HADOOP_HOME/libexec/etc/hadoop"
export SPARK_HOME="/usr/local/Cellar/apache-spark/1.1.0"
```

Now we need to reload the environment for our changes to take effect
add bin folders to path variable

```
source ~/.bash_profile
```

Add Hadoop and Spark bin directories to the class path

```
export PATH="$PATH":$HADOOP_HOME/bin:$SPARK_HOME/bin
```

Configure passphraseless SSH on localhost

```
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

#### Configure Hadoop

The following instructions are to configure Hadoop as a single-node in a pseudo-distributed mode with MapReduce job execution on YARN.

```
cd usr/local/Cellar/hadoop/2.6.0/libexec/
```

Edit ‘etc/hadoop/hadoop-env.sh':

```
# this fixes the "scdynamicstore" warning
export HADOOP_OPTS="$HADOOP_OPTS -Djava.security.krb5.realm= -Djava.security.krb5.kdc="
```

Add the following configuration settings:

In file ‘etc/hadoop/mapred-site.xml':

```
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

In file ‘etc/hadoop/yarn-site.xml':

```
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```

In file ‘etc/hadoop/core-site.xml':

```
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
</configuration>
```

In file ‘etc/hadoop/hdfs-site.xml':

```
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
</configuration>
```


#### Start Hadoop

Start by moving to the root directory

```
cd /usr/local/Cellar/hadoop/2.6.0
```

Format the Hadoop HDFS filesystem (be very careful if you have an existing Hadoop installation)

```
./bin/hdfs namenode -format
```

Start the NameNode daemon & DataNode daemon

```
./sbin/start-dfs.sh
```

Browse the web interface for the NameNode

```
http://localhost:50070/
```

Start ResourceManager daemon and NodeManager daemon:

```
./sbin/start-yarn.sh
```

Browse the web interface for the ResourceManager

```
http://localhost:8088/
```

Create the HDFS directories required to execute MapReduce jobs:

```
./bin/hdfs dfs -mkdir -p /user/{username}
```

#### Start Spark 

Move to the Spark directory

```
cd /usr/local/Cellar/apache-spark/1.1.0
```

To start the spark shell using YARN 

```
./bin/spark-shell --master yarn

println("Hello Spark Shell on YARN")
```

#### Standalone Application

Navigate to your workspace destination of choice and create a new scala file 

```
cd development/spark/

mkdir hello_world

vim HelloWorldJob.scala
```

Add the following content to HelloWorldJob.scala

```
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.SparkConf
 
object HelloWorld {
  def main(args: Array[String]) {
 
   val conf = new SparkConf().setMaster("local")
                             .setAppName("HelloWorld")
                             
   val spark = new SparkContext(conf)
 
   println("Hello, world!")
 
   spark.stop()
 
  }
}
```

We now need to add a build sbt file which will tell the sbt build engine what dependencies etc. to import and some basic project information.

```
vim build.sbt
```

Add the following code to the build.sbt file.

```
name := "Hello World"

version := "1.0"

scalaVersion := "2.10.4"

libraryDependencies += "org.apache.spark" %% "spark-core" % "1.1.1"
```

Now to build and package the jar using sbt

```
mkdir -p src\main\scala
mv HelloWorldJob.scala !$ 
sbt clean compile package
```

Now time to run the jar on YARN 

```
spark-submit --class HelloWorld --master yarn-cluster target/scala-2.10/helloworld_2.10-1.0.jar
```

Every spark context can be monitored in the browser by navigating to 

```
http://localhost:4040/
```

You should now see the HelloWorld job which will either have a status of executing or complete if it had already run. There are also link to the log for the job which you can also view in the browser. 

