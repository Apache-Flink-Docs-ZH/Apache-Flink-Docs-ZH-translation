---
title: "Generating Timestamps / Watermarks"
nav-parent_id: event_time
nav-pos: 1
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

* toc
{:toc}

本章节与运行在 **事件时间 (event time)** 的程序相关。想要了解关于 *事件时间 (event time)* , *处理时间 (processing time)*, 和 *摄入时间 (ingestion time)*, 请参阅 [事件时间介绍]({{ site.baseurl }}/dev/event_time.html) 。

如果需要使用 *事件时间*, 流程序需要设置对应的 *time characteristic* 。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
{% endhighlight %}
</div>
</div>

## 分配时间戳 (Timestamps)

如果要用 *事件时间* 工作， Flink 需要知道事件的 *时间戳*， 这意味着流中的每一个元素需要 *分配* 到自己的时间时间戳。 这实际上通过从元素的某个字段抽取 / 访问得到。

伴随着时间戳的分配水位 (watermarks) 会不断产生， 这能告知系统事件时间的进度。

有两个方式可以分配时间戳和产生水位：

  1. 直接在数据流的源中
  2. 通过一个时间戳分配器 / 水位产生器 (timestamp assigner / watermark generator): 在 Flink时间戳分配器中还可以定义水位发射给 Flink

<span class="label label-danger">注意</span> 时间戳和水位均以微妙为单位指定，因为 Java 的起始时间为 1970-01-01T00:00:00Z 。

### 带时间戳和水位的源函数

数据流的源也能直接分配时间戳给产生的元素， 并且也可以发射水位。 当这些都被做完之后， 就不需要时间戳分配器了。
注意， 如果使用了一个时间戳分配器， 任何通过源提供的时间戳和水位会被覆盖。

为了给源里的一个元素直接分配一个时间戳， 源必须使用在 `SourceContext` 上的 `collectWithTimestamp(...)` 方法。 为了产生水位， 源必须调用 `emitWatermark(Watermark)` 函数。

以下是一个分配时间抽和产生水位的 *非检查点(non-checkpointed)* 源的简单例子：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
override def run(ctx: SourceContext[MyType]): Unit = {
	while (/* condition */) {
		val next: MyType = getNext()
		ctx.collectWithTimestamp(next, next.eventTimestamp)

		if (next.hasWatermarkTime) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime))
		}
	}
}
{% endhighlight %}
</div>
</div>


### 时间戳分配器 / 水位产生器 (Timestamp Assigners / Watermark Generators)

时间戳分配器会从一个流生成一个新的带有时间戳元素的水位的流。 如果原来的流已经有时间戳和 / 或水位， 时间戳分配器会将它们复写。

时间戳分配器经常在数据源之后立即被指定， 但不严格要求一定要这么做。 举例来说， 一个通用的模式是先解析 (*映射函数 (MapFunction)*) 并过滤 (*过滤函数 (FilterFunction)*) 再使用时间分配器。 无论如何， 时间戳分配器需要在事件时间上的第一个操作 (比如第一个窗口操作 (window operation)) 之前指定。 作为一个特例， 当使用 Kafka 作为一个流式作业的源时， Flink 允许在源 (或者消费者) 内指定一个时间戳分配器 / 水位产生器。 更多关于如何这么做的信息可以参阅 [Kafka连接器文档]({{ site.baseurl }}/dev/connectors/kafka.html)。

**注意:** 该章节剩下的部分会展示编程人员必须实现的主要接口来创建自己的时间戳抽取器 / 水位发射器。 如果要看发送给Flink的预实现抽取器， 请参阅 [预定义水位抽取器 / 水位发射器]({{ site.baseurl }}/dev/event_timestamp_extractors.html) 页面。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

DataStream<MyEvent> withTimestampsAndWatermarks = stream
        .filter( event -> event.severity() == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks());

withTimestampsAndWatermarks
        .keyBy( (event) -> event.getGroup() )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) -> a.add(b) )
        .addSink(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream: DataStream[MyEvent] = env.readFile(
         myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
         FilePathFilter.createDefaultFilter());

val withTimestampsAndWatermarks: DataStream[MyEvent] = stream
        .filter( _.severity == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks())

withTimestampsAndWatermarks
        .keyBy( _.getGroup )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) => a.add(b) )
        .addSink(...)
{% endhighlight %}
</div>
</div>


#### **使用周期水位 (Periodic Watermarks)**

`AssignerWithPeriodicWatermarks` 分配时间抽并周期性产生水位 (可能根据流中的数据， 或纯粹根据处理时间 (processing time))。

水位产生的间隔 (每 *n* 毫秒) 通过 `ExecutionConfig.setAutoWatermarkInterval(...)` 定义。 分配器的 `getCurrentWatermark()` 方法每次间隔后都会被调用， 如果新返回的水位是非空并且大于之前的水位， 该水位会被发射给 Flink。

周期产生水位的时间戳分配器的两个简单例子如下所示。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
/**
 * 假设元素到达的时候某种程度是乱序的， 该生成器生成水位。 
 * 最近的带有时间戳 t 的元素最多会在最早带该时间戳 t 的元素之后的 n 毫秒内到达。
 */
public class BoundedOutOfOrdernessGenerator extends AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

/**
 * 该生成器产生滞后于处理时间一个固定间隔的水位。
 * 该生成器假设元素在一个有限的延迟之后到达 Flink 中。
 */
public class TimeLagWatermarkGenerator extends AssignerWithPeriodicWatermarks<MyEvent> {

	private final long maxTimeLag = 5000; // 5 seconds

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark getCurrentWatermark() {
		// return the watermark as current time minus the maximum time lag
		return new Watermark(System.currentTimeMillis() - maxTimeLag);
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
/**
 * 假设元素到达的时候某种程度是乱序的， 该生成器生成水位。 
 * 最近的带有时间戳 t 的元素最多会在最早带该时间戳 t 的元素之后的 n 毫秒内到达。
 */
class BoundedOutOfOrdernessGenerator extends AssignerWithPeriodicWatermarks[MyEvent] {

    val maxOutOfOrderness = 3500L; // 3.5 seconds

    var currentMaxTimestamp: Long;

    override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
        val timestamp = element.getCreationTime()
        currentMaxTimestamp = max(timestamp, currentMaxTimestamp)
        timestamp;
    }

    override def getCurrentWatermark(): Watermark = {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

/**
 * 该生成器产生滞后于处理时间一个固定间隔的水位。
 * 该生成器假设元素在一个有限的延迟之后到达 Flink 中。
 */
class TimeLagWatermarkGenerator extends AssignerWithPeriodicWatermarks[MyEvent] {

    val maxTimeLag = 5000L; // 5 seconds

    override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
        element.getCreationTime
    }

    override def getCurrentWatermark(): Watermark = {
        // return the watermark as current time minus the maximum time lag
        new Watermark(System.currentTimeMillis() - maxTimeLag)
    }
}
{% endhighlight %}
</div>
</div>

#### **使用点断水位 (Punctuated Watermarks)**

为了在一个暗示可能会产生一个新的水位的事件发生时产生水位， 使用 `AssignerWithPunctuatedWatermarks`。 对于这个类， Flink会先调用 `extractTimestamp(...)` 方法给元素分批恶一个时间抽， 接着立即在该元素上调用 `checkAndGetNextWatermark(...)` 方法。

该 `checkAndGetNextWatermark(...)` 方法会传入在 `extractTimestamp(...)` 方法中分配的时间戳， 并决定该时间抽是否产生一个水位。 当 `checkAndGetNextWatermark(...)` 方法返回一个非空水位， 并且水位比先前最近的水位大， 这个新水位会被发射给 Flink。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks[MyEvent] {

	override def extractTimestamp(element: MyEvent, previousElementTimestamp: Long): Long = {
		element.getCreationTime
	}

	override def checkAndGetNextWatermark(lastElement: MyEvent, extractedTimestamp: Long): Watermark = {
		if (lastElement.hasWatermarkMarker()) new Watermark(extractedTimestamp) else null
	}
}
{% endhighlight %}
</div>
</div>

*Note:* It is possible to generate a watermark on every single event. However, because each watermark causes some
computation downstream, an excessive number of watermarks degrades performance.


## Timestamps per Kafka Partition

When using [Apache Kafka](connectors/kafka.html) as a data source, each Kafka partition may have a simple event time pattern (ascending
timestamps or bounded out-of-orderness). However, when consuming streams from Kafka, multiple partitions often get consumed in parallel,
interleaving the events from the partitions and destroying the per-partition patterns (this is inherent in how Kafka's consumer clients work).

In that case, you can use Flink's Kafka-partition-aware watermark generation. Using that feature, watermarks are generated inside the
Kafka consumer, per Kafka partition, and the per-partition watermarks are merged in the same way as watermarks are merged on stream shuffles.

For example, if event timestamps are strictly ascending per Kafka partition, generating per-partition watermarks with the
[ascending timestamps watermark generator](event_timestamp_extractors.html#assigners-with-ascending-timestamps) will result in perfect overall watermarks.

The illustrations below show how to use ther per-Kafka-partition watermark generation, and how watermarks propagate through the
streaming dataflow in that case.


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
FlinkKafkaConsumer09<MyType> kafkaSource = new FlinkKafkaConsumer09<>("myTopic", schema, props);
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyType>() {

    @Override
    public long extractAscendingTimestamp(MyType element) {
        return element.eventTimestamp();
    }
});

DataStream<MyType> stream = env.addSource(kafkaSource);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val kafkaSource = new FlinkKafkaConsumer09[MyType]("myTopic", schema, props)
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor[MyType] {
    def extractAscendingTimestamp(element: MyType): Long = element.eventTimestamp
})

val stream: DataStream[MyType] = env.addSource(kafkaSource)
{% endhighlight %}
</div>
</div>

<img src="{{ site.baseurl }}/fig/parallel_kafka_watermarks.svg" alt="Generating Watermarks with awareness for Kafka-partitions" class="center" width="80%" />


