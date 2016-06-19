---
layout: post
title: Spark Streaming Rate Limiting and Back Pressure
---

With the release of Spark 1.6 came the feature to allow your streaming applications to apply back pressure, which is a form of rate limiting that ensures that your streaming application can handle spikes in events without flooding your clusters resources and causing batches to back up.

Spark streaming can be broken down into two components, a receiver, and the processing engine. The receiver can come in many forms, you can use one of the built in and popular receivers such as the one for Kafka or Kinesis to a simple socket 
receiver which merely reads data from a socket. The receiver will iterate until it is killed reading data over the network from one of the input sources listed above, the data is then written to the block manager by calling the store function.

The receiver builds blocks of data which the Spark Streaming computation engine will read in in the form of a DStream, which in simple terms is just a constance stream of RDDs. The rate at which the engine (your business logic) reads in the data is set by the batchWindow. For example if it is set to 10 seconds it will try to read in the number of blocks that represents 10 seconds. By default the rate at which the receiver builds blocks is 200ms so for a 10 second window the amount of block read in will be 50, this will mean that the DStream will have 50 partitions of data. 

I will hopefully, go through this in a later blog post in more detail.

### Simple Streaming Computation

```
// simple computation on the receiver DStream

val dataStream = ReceiverUtil.CreateStream()

dataStream.foreachRDD { rdd =>
	println(s"processed ${rdd.count} data")
}
```

## The Problem

If the receiver is writing blocks of data quicker than the processing engine can process the data blocks then you can end up with a lot of batches backed up and quickly fall behind. Also if the amount of data that is being written to one block increases then the processing engine can start to struggle and the processing time will increase which has the knock on effect of increasing the scheduling delay and processing delay, all of which make for an unstable streaming application. 

## The Solution

At first glance finding how to implement these newly added features was suprisingly more difficult than you may think. It took 
a lot of trial and error and searching through the Spark source code in order to find out exactly which dials I needed to turn in 
order to get a more stable streaming application; Below are my findings

### Receiver Max Rate

My first port of call was to add `spark.streaming.receiver.maxRate` which sets the maximum rate at which each receiver should write to the block store. As of the time of writing this the lowest rate you can set this too is 100.

```
conf.set("spark.streaming.receiver.maxRate", "100")
```

### Enable Backpressure

I then enabled back pressure using the setting `spark.streaming.backpressure.enabled` and waited for the magic to happen... it didn't. Using a custom receiver and enabling this setting and the one above didn't really get us anything apart from an upper bound for each receiver, which is fine as long as your clusters resources don't get swamped and a max rate of 100 really isn't going to cut it. 

```
conf.set("spark.streaming.backpressure.enabled", "true")
```

### Rate Estimator

Enter the Rate Limiter. Deep within the spark source code there is a PIDRateEstimator which will calculate the rate at which the receiver will write to blocks to the block manager, this is done using a standard algorithm for rate calculation [Add Wiki link]  using the scheduling delay, processing time and some other factors to calculate if we are receiving data at a rate that is higher than we can process. If the rate is to high then it is ajusted to a new rate, if the rate is to low then the rate is increased.

This sounded to good to be true and indeed it was, the code is all marked as private and is not exposed by the developer API. In order to get around this fact you have to mark your InputDStream as being part of the org.apache.spark package and then implement your own Rate Controller within your customm ReceiverInputDStream class.

```

case class Order(customerId: Int, productId: Int, price: Double, quantity: Int)

class OrderInputDStream[T: ClassTag](
    _ssc: StreamingContext,
    host: String,
    port: Int,
    parallelism: Int,
    storageLevel: StorageLevel
) extends ReceiverInputDStream[Order](_ssc) {

  override def getReceiver(): Receiver[Order] = {
    new OrderReceiver(host, port, parallelism, storageLevel)
  }

   /**
   * Asynchronously maintains & sends new rate limits to the receiver through the receiver tracker.
   */
  override protected[streaming] val rateController: Option[RateController] = {
    if (RateController.isBackPressureEnabled(ssc.conf)) {
      Some(new OrderRateController(id,
        RateEstimator.create(ssc.conf, context.graph.batchDuration)))
    } else {
      None
    }
  }

  /**
   * A RateController to retrieve the rate from RateEstimator.
   */
  private[streaming] class OrderRateController(id: Int, estimator: RateEstimator)
    extends RateController(id, estimator) {
    override def publish(rate: Long): Unit = ()
  }
}

```

Another feature I found really useful was turning on the rate estimator trace logging, this gives you a detailed trace in your logs and on the console of how the rate is being estimated. In the later example you will see how the rate increases when we have spare capacity and how the rate is lowered when we get a spike in traffic which we cannot keep up with.

```
Logger.getLogger("org.apache.spark.streaming.scheduler.rate.PIDRateEstimator").setLevel(Level.TRACE) 
```

There are a few few settings that can be tweaked in order to make the rate estimater more or less aggressive, depending on your workload. This will take trial and error on your part to get a nice balancer that gives your maximum through put with good stability.

```
conf.set("spark.streaming.backpressure.pid.minRate", "1.0")
conf.set("spark.streaming.backpressure.pid.proportional", "1.0")
conf.set("spark.streaming.backpressure.pid.derived", "0.0")
conf.set("spark.streaming.backpressure.pid.integral", "0.2")
```
