---
layout: page
title: Metrics
---

Colossus uses the (currently nameless) metrics library.

## Introduction

High-throughput Colossus services can serve hundreds of thousands of requests
per second, which can easily translate to millions of recordable events per
second.  The Metrics library provides a way to work with metrics with as little
overhead as possible.

The metrics library performs 3 important operations

* **Event Collection** : providing a way for user-code to signal that something worth tracking has happened
* **Metrics Aggregation** : collecting raw events into metrics that can be filtered and aggregated in real-time
* **Reporting** : packaging up and sending data to some external database (such as OpenTSDB)

## Basic Architecture

The heart of metrics is a `MetricSystem`, which is a set of actors that handle
all of the background operations of dealing with metrics.  In most cases, you
only want to have one `MetricSystem` per application.

Metrics are generated periodically by a `Tick` message published on the global
event bus.  By default this happens once per second, but it can be configured
to any time interval.  So while events are being collected as they occur,
compiled metrics (such as rates and histogram percentiles) are generated once
per tick.

### Structure of a Metric

Every metric in a system has a unique name with a url-like structure.  For
example, `/my-service/requests` and `/my-service/system/gc/msec` are two
metrics automatically generated by Colossus.  Every metric contains a set of
integer values, with each value uniquely identified by a set of key/value tags.
For example, the values for the `system/gc/msec` metric might look like

{% highlight scala %}

/my-service/system/gc/msec
  [type = ParNew] : 123456
  [type = ConcurrentMarkSweep] : 9876543


{% endhighlight %}

Tags are immensely useful when required more fine-grained breakdown of
imformation.  A metric value can have multiple tags, values in a single metric
do not all need the same tags, and tags are optional.  We will see later that
tags can also be used to filter and aggregate values.


## Getting Started

If you are using colossus, it depends on the metrics library and pulls it in.  Otherwise you must add the following to your build.sbt/Build.scala

{% highlight scala %}

libraryDependencies += "com.tumblr" %% "metrics" % "0.2.0"

{% endhighlight %}

### Quickstart

Here's a quick demo of using the metrics system.

{% highlight scala %}

import akka.actor._
import metrics._
import scala.concurrent.duration._

implicit val actor_system = ActorSystem()

//create the metric system
val metric_system = MetricSystem("/my-service")

//get a collection
val collection = metric_system.sharedCollection

//now get a rate from the collection
val rate = collection getOrAdd Rate("/my-rate", periods = List(1.second, 1.minute))

//fire off some events!
rate.hit()
rate.hit(25)

//by default, metrics are aggregated and snapshotted once per second
Thread.sleep(1000)

//the snapshot is sent to an akka agent in the metric_system
println(metric_system.snapshot()("/my-service/my-rate")())

{% endhighlight %}




## Event Collection

In most cases, metrics are generated from various events triggered by
application code.  For example, we can keep track of a service's request rate
by firing an event every time a request finishes processing.  Likewise, we can
keep track of latency by adding the proccessing time of each request to a
histogram.

Event Collectors take the role of providing a simple API for generating events.

There are currently 4 built-in event collectors:

* Counter - keeps track of a single number which can process increment/decrement operations
* Rate - A rate can be "hit" and it will track hits/second (or other time periods)
* Histogram - Similar to a rate, but each hit includes an integer value.  The histogram will generate percentiles/min/max over periods of time
* Gauge - set a static integer value

Each metric has numerous configuration options to ensure the best measurements for the job.


### Local vs Shared Collectors

The tl;dr is, shared collectors are backed by actors while local collectors are
used inside actors.  Shared collectors are thread-safe, but since each event is
an actor message, have a bit of overhead at high frequency.  Local collectors
are not thread safe, but have almost no overhead.  

Every local collector can easily be converted into a shared collector by
calling its `shared` method, and both version implement the same base interface
so the API is the same.

So if you are writing actors and want to track events inside them, your best
bet is to extends the `ActorMetrics` trait.  For example:

{% highlight scala %}

class MyActor(val metricSystem: MetricSystem) extends Actor with ActorMetrics {
  
  val rate = metrics getOrAdd Rate("/foos")

  def receive = handleMetrics orElse {
    case "Foo" => {
      rate.hit()
    }
    case "FOOOO" => Future { 
      rate.shared.hit() //thread-safe
    }
  }

}

{% endhighlight %}

## Metric Snapshots and Aggregation

As mentioned before, the Metric system will generate a snapshot once per tick,
which defaults to once per second.  The easiest way to access this snapshot is
through the `snapshot` field on the `MetricSystem`.  This is an akka agent that
contains the most recent snapshot.

Notice that what values actually appear in the snapshot depend on how your
event collectors are configured.  For example, if you configure a rate to
aggregate in events per minute, the rate will only update its value once per
minute regardless of the frequency configured for snapshotting.  

## Metric Reporting

Currently metric reporting is mostly focused on reporting to OpenTSDB.  To setup reporting you basically need 2 things:

* A MetricSender - this is the object that encodes metrics to be sent
* A set of metric filters - These are used to select and aggregate which metrics to send

