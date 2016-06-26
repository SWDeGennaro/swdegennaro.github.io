---
layout: post
title: Connecting to Spark via JDBC using the Thrift server
---

I've recently been investigating how to make Apache Spark a lot more user friendly and ways in which you can harness the power of Spark whilst still keeping traditional ETL work flows. What do I mean by a traditional ETL work flow? In simple terms an ordered set of SQL scripts, a script runner and finally a scheduler.

So the aim was to be able to run a SQL script from my local machine on a cluster. After some research it seemed that my best option would be to run the Thrift server which comes with Spark on EMR. Once the thrift server is running it alows you to connect to Hive via JDBC and run HiveQL quries on top of Apache Spark. 

### Starting the Thrift server

#### SSH into master node
```
ssh hadoop@${EMR_MASTER}
```

#### start the Thrift server
```
sudo -u hadoop /usr/lib/spark/sbin/start-thriftserver.sh --master yarn-client --num-executors 5 --executor-cores 15 --executor-memory 28G --hiveconf hive.server2.thrift.port=10001
```

#### test the connection on the cluster 
```
/usr/lib/spark/bin/beeline -u 'jdbc:hive2://localhost:10001/' -n hadoop
```

#### ssh tunnel to master node forwarding localhost:10001 to master thrift port (10001)
```
ssh -N -L 10001:localhost:10001 hadoop@${EMR_MASTER}
```

#### connect to beeline from local machine this will be forwarded to master node
```
beeline -u 'jdbc:hive2://127.0.0.1:10001' -n hadoop
```

### An example

In order to prove the concept I will put a small file into HDFS and then attempt to query it from a SQL file that I will execute via an JDBC from my local machine. 

```
cat logs.txt

pppa006.compuserve.com	-	807256800	GET	/images/launch-logo.gif	200	1713		
vcc7.langara.bc.ca	-	807256804	GET	/shuttle/missions/missions.html	200	8677		
pppa006.compuserve.com	-	807256806	GET	/history/apollo/images/apollo-logo1.gif	200	1173		
thing1.cchem.berkeley.edu	-	807256870	GET	/shuttle/missions/sts-70/sts-70-day-03-highlights.html	200	4705		
202.236.34.35	-	807256881	GET	/whats-new.html	200	18936		
bettong.client.uq.oz.au	-	807256884	GET	/history/skylab/skylab.html	200	1687		
202.236.34.35	-	807256884	GET	/images/whatsnew.gif	200	651		
202.236.34.35	-	807256885	GET	/images/KSC-logosmall.gif	200	1204		
bettong.client.uq.oz.au	-	807256900	GET	/history/skylab/skylab.html	304	0		
bettong.client.uq.oz.au	-	807256913	GET	/images/ksclogosmall.gif	304	0		
bettong.client.uq.oz.au	-	807256913	GET	/history/apollo/images/apollo-logo.gif	200	3047		
hella.stm.it	-	807256914	GET	/shuttle/missions/sts-70/images/DSC-95EC-0001.jpg	200	513911		
mtv-pm0-ip4.halcyon.com	-	807256916	GET	/shuttle/countdown/	200	4324	
```

#### put the file into hdfs
```
hadoop fs -mkdir /user/hadoop/dir1
hadoop fs -put logs.txt /user/hadoop/dir1/logs.txt
```

#### test query in test_query.sql

```

CREATE TEMPORARY TABLE t1 (
  host			STRING,
  logname		BIGINT,
  time			TIMESTAMP,
  method		STRING,
  url			STRING
  response		INT,
  bytes			BIGINT,
  referer		STRING,
  useragent		STRING
);

LOAD DATA INPATH '/user/hadoop/dir1/logs.txt' INTO TABLE t1

CACHE TABLE t1

SELECT url, count(*) c
FROM t1
GROUP BY url ORDER BY c DESC LIMIT 10;

UNCACHE TABLE t1;
```

#### execute the sql file
```
beeline -u 'jdbc:hive2://127.0.0.1:10001' -n hadoop -f test_query.sql
```
