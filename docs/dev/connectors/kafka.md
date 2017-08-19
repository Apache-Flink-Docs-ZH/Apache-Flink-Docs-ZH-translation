---
title: "Apache Kafka Connector"
nav-title: Kafka
nav-parent_id: connectors
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

* This will be replaced by the TOC
{:toc}

该连接器为 [Apache Kafka](https://kafka.apache.org/) 服务的事件流提供接入。

Flink 提供特别的 Kafka 连接器来从 Kafka 主题 (topic) 读数据或写数据到 Kafka 主题。 Flink 的 Kafka 消费者整合 Flink 的记录点 (checkpointing) 机制来提供正好一次处理语义 (exactly-once processing semantics)。 为了将其实现， Flink 不仅依靠 Kafka 的消费者群体偏移追踪 (group offset tracking)， 还在内部追踪并记录 (checkpoint) 这些偏移 (offset)。

请为你的使用情况和环境选择一个包 (maven arteifact id) 和类名。
对于大多数用户， `FlinkKafkaConsumer08` (`flink-connector-kafka` 的一部分) 是合适可用的。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">Maven Dependency</th>
      <th class="text-left">Supported since</th>
      <th class="text-left">Consumer and <br>
      Producer Class name</th>
      <th class="text-left">Kafka version</th>
      <th class="text-left">Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer08<br>
        FlinkKafkaProducer08</td>
        <td>0.8.x</td>
        <td>Uses the <a href="https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example">SimpleConsumer</a> API of Kafka internally. Offsets are committed to ZK by Flink.</td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.9{{ site.scala_version_suffix }}</td>
        <td>1.0.0</td>
        <td>FlinkKafkaConsumer09<br>
        FlinkKafkaProducer09</td>
        <td>0.9.x</td>
        <td>Uses the new <a href="http://kafka.apache.org/documentation.html#newconsumerapi">Consumer API</a> Kafka.</td>
    </tr>
    <tr>
        <td>flink-connector-kafka-0.10{{ site.scala_version_suffix }}</td>
        <td>1.2.0</td>
        <td>FlinkKafkaConsumer010<br>
        FlinkKafkaProducer010</td>
        <td>0.10.x</td>
        <td>This connector supports <a href="https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message">Kafka messages with timestamps</a> both for producing and consuming.</td>
    </tr>
  </tbody>
</table>

接着， 把连接器导入到你的 maven 项目中：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.8{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

注意到流连接器 (streaming connectors) 目前为不属于 binary distribution 的一部分。 点击 [这里]({{ site.baseurl}}/dev/linking.html) 查看如何将它们与云端链接执行。

## 安装 Apache Kafka

* 遵循 [Kafka's 快速入门](https://kafka.apache.org/documentation.html#quickstart) 的指导来下载代码并启动服务器 (必须在开始一个应用之前启动 ZooKeeper 和 Kafka 服务器)。
* 如果 Kafka 和 ZooKeeper 服务器在远程机器上运行， 则 `config/server.properties` 文件中的 `advertised.host.name` 配置必须设为该机器的 IP 地址。

## Kafka 消费者 (Consumer)

Flink 的 Kafka 消费者为 `FlinkKafkaConsumer08` (或对于 Kafka 0.9.0.x 版本是 `09` 等)。 它提供了对一个或多个 Kafka 主题的接入。

其构造器接收一下参数：

1. 主题名 / 主题名列表
2. 一个 DeserializationSchema / KeyedDeserializationSchema 来反序列化来自 Kafka 的数据
3. Kafka 消费者的属性
  以下属性是必须的：
  - "bootstrap.servers" (若有多个 Kafka 中间者 (broker)， 用逗号隔开)
  - "zookeeper.connect" (若有多个 Zookeeper 服务器， 用逗号隔开) (**仅在 Kafka 0.8 中是必须的**)
  - "group.id" 消费者群体 (Consumer Group) 的 ID

比如:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
DataStream<String> stream = env
	.addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
stream = env
    .addSource(new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties))
    .print
{% endhighlight %}
</div>
</div>

当前 FlinkKafkaConsumer 的实现会建立一个来自客户端的连接 (当调用构造器时) 来查询主题列表和分区 (partition)。

要让该例子工作，该消费者需要能从提交作业到 Flink 集群的及其访问消费者。
如果你在 Kafka 消费者的客户端遇到任何问题， 可以在客户端日志中查看关于失败请求等问题的信息。

### `DeserializationSchema`

Flink 的 Kafka 消费者需要直到如何把 Kafka 内的二元数据变成 Java/Scala 对象。 `DeserializationSchema` 允许用户指定这样一个 schema。 
Flink 会为每条消息调用 `T deserialize(byte[] message)` 方法， 将来自 Kafka 的消息传进去。

一般情况下从 `AbstractDeserializationSchema` 开始是比较有助的， 该类负责为 Flink 的类型系统描述所产生的 Java/Scala 类型。 实现标准的 `DeserializationSchema` 的用户需要实现 `getProducedType(...)` 方法。

为了访问 Kafka 信息的键和值， `KeyedDeserializationSchema` 有一个反序列化方法 ` T deserialize(byte[] messageKey, byte[] message, String topic, int partition, long offset)`。

为了方便用户， Flink提供下列 schemas：

1. `TypeInformationSerializationSchema` (和 `TypeInformationKeyValueSerializationSchema`)， 该类根据 Flink 的 `TypeInformation` 
    创建一个 schema， 如果数据是由 Flink 的读写的， 该类非常有用。 这个 schema 是 Flink 专属的泛型序列化方法。
    
2. `JsonDeserializationSchema` (and `JSONKeyValueDeserializationSchema`)， 该类能将 JSON 序列化成一个 ObjectNode 对象， 通过该类可使
    用 objectNode.get("field").as(Int/String/...)() 方法访问字段。 键值对形式的 objectNode 包含一个 "键" 和 "值" 字段， 它们包含了所有的
    字段和暴露消息的偏移/分区/主题的可选的 "元数据" 字段。 

当遇到任何理由引起的无法被反序列化的坏消息， 有两种处理方法 - 可以选择从 `deserialize(...)` 方法抛出异常， 这样会引起作业失败和重启， 或者选择返回 `null` 来允许 Flink Kafka 消费者安静地跳过坏消息。 注意到由于消费者的容错机制 (可参见以下章节获取更详细的信息)， 作业在坏消息上的失败会让消费者再次尝试反序列化消息。 因此 如果反序列化仍然失败， 消费者会一直重启并陷入反序列化坏消息的循环。

### Kafka 消费者起始位置配置

Flink Kafka 消费者允许用户通过配置决定 Kafka 分区的起始位置。

比如:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

FlinkKafkaConsumer08<String> myConsumer = new FlinkKafkaConsumer08<>(...);
myConsumer.setStartFromEarliest();     // start from the earliest record possible
myConsumer.setStartFromLatest();       // start from the latest record
myConsumer.setStartFromGroupOffsets(); // the default behaviour

DataStream<String> stream = env.addSource(myConsumer);
...
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val myConsumer = new FlinkKafkaConsumer08[String](...)
myConsumer.setStartFromEarliest()      // start from the earliest record possible
myConsumer.setStartFromLatest()        // start from the latest record
myConsumer.setStartFromGroupOffsets()  // the default behaviour

val stream = env.addSource(myConsumer)
...
{% endhighlight %}
</div>
</div>

所有版本的 Kafka 消费者都有上述配置方法来设置起始位置。

 * `setStartFromGroupOffsets` (默认行为): 从 Kafka 中间者中 (如果是Kafka 0.8 则为 ZooKeeper) 消费者群体提交的偏移量 (消费者属性中设置的 `group.id` ) 开始读分区。 如果不能找到一个分区的偏移量， 属性中的 `auto.offset.reset` 会被使用。
 * `setStartFromEarliest()` / `setStartFromLatest()`: 从最早 / 最近的记录开始。 如果使用该方法 Kafka 中提交的偏移量会被忽略， 并且不会作为
 起始位置被使用。 
 
你也能为每个分区直接指定起始的偏移量：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Map<KafkaTopicPartition, Long> specificStartOffsets = new HashMap<>();
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L);

myConsumer.setStartFromSpecificOffsets(specificStartOffsets);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val specificStartOffsets = new java.util.HashMap[KafkaTopicPartition, java.lang.Long]()
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L)
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L)

myConsumer.setStartFromSpecificOffsets(specificStartOffsets)
{% endhighlight %}
</div>
</div>

上述例子为 `myTopic` 主题的0号， 1号， 2号分区指定起始偏移量。 该偏移量是消费者在每个分区要读的下一条记录。 注意到如果消费者需要读一个在提供的偏移量映射中没有指定偏移量的分区， 它会对这个特别的分区使用默认的群体偏移量行为 (即 `setStartFromGroupOffsets()`)

需要注意的是这些起始位置配置方法在作业从失败中自动恢复或使用保存点认为恢复时不会影响起始位置。 在恢复时， 每个 Kafka 分区的起始位置有保存在保存点或记录点的偏移量决定 (请参阅下一章节了解关于通过记录点启动消费者容错机制的信息)。

### Kafka 消费者和容错机制

当 Flink 的启用记录点时， Flink Kafka 消费者会从一个主题中消费记录，并用一致的方式周期性记录所有 Kafka 偏移量和其它算子的状态。 当作业失败时， Flink会将流程序恢复到最忌的记录点并重新从Kafka消化数据， 重新消化时会从保存在记录点的偏移量开始消化。

记录点的间隔定义了程序在作业失败时最多从多远的时间点恢复。

如果要使用能容错的 Kafka 消费者， 拓扑图的记录点功能需要在治病环境中启用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(5000); // checkpoint every 5000 msecs
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.enableCheckpointing(5000) // checkpoint every 5000 msecs
{% endhighlight %}
</div>
</div>

需要注意的是， Flink 只会在有足够的处理分片 (processing slot) 时才会重启拓扑图。 所以如果拓扑图在因为 TaskManager 的丢失而失败时， 依旧需要保证有足够的分片来进行重启。 运行在 YARN 上的 Flink 支持自动重启丢失的 YARN 容器。

如果记录点没有启用， Kafka 消费者会周期性向 ZooKeeper 提交偏移量。

### Kafka 消费者偏移量提交行为配置

Flink Kafka 消费者允许配置偏移量提交到 Kafka 中间者 (或 ZooKeeper 在 0.8 版本) 的行为。 注意到 Flink Kafka 消费者不依赖提交的偏移量来保证容错。 提交的偏移量只是一种出于监控 (monitoring) 目的揭露消费者进度的方法。

配置偏移量提交行为的方式根据记录点是否启动而有所不同。

 - *记录点不启动 (checkpointing disabled):* 如果记录点没有启动， Flink Kafka 消费者依赖内部使用的 Kafka 客户端的周期性偏移量自动提交功能。 因
 此， 如果要关闭或启动偏移量提交， 只需简单地在提供的 `Properties` 配置中为 `enable.auto.commit` (或 `auto.commit.enable` 在 Kafka 0.8 中) 
 / `auto.commit.interval.ms` 设置合适的值即可。
 
 - *记录点启动 (Checkpointing enabled):* 如果记录点启动， Flink Kafka 消费者会在记录完成时把偏移量提交到记录的状态中保存。 这确保了在 Kafka 中
 间者提交的偏移量与记录状态中的偏移量是一致的。 用户能选择通过调用消费者上的 `setCommitOffsetsOnCheckpoints(boolean)` 方法关闭或启用偏移量提交
 (默认情况下 该行为为 `true`)。
 需要注意的是在这种情况下， `Properties` 中的周期性偏移量自动提交设定会被完全忽略。

### Kafka 消费者和时间戳抽取/水位发射

In many scenarios, the timestamp of a record is embedded (explicitly or implicitly) in the record itself.
In addition, the user may want to emit watermarks either periodically, or in an irregular fashion, e.g. based on
special records in the Kafka stream that contain the current event-time watermark. For these cases, the Flink Kafka
Consumer allows the specification of an `AssignerWithPeriodicWatermarks` or an `AssignerWithPunctuatedWatermarks`.

You can specify your custom timestamp extractor/watermark emitter as described
[here]({{ site.baseurl }}/apis/streaming/event_timestamps_watermarks.html), or use one from the
[predefined ones]({{ site.baseurl }}/apis/streaming/event_timestamp_extractors.html). After doing so, you
can pass it to your consumer in the following way:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer08<String> myConsumer =
    new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties);
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter());

DataStream<String> stream = env
	.addSource(myConsumer)
	.print();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");

val myConsumer = new FlinkKafkaConsumer08[String]("topic", new SimpleStringSchema(), properties);
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter());
stream = env
    .addSource(myConsumer)
    .print
{% endhighlight %}
</div>
</div>

Internally, an instance of the assigner is executed per Kafka partition.
When such an assigner is specified, for each record read from Kafka, the
`extractTimestamp(T element, long previousElementTimestamp)` is called to assign a timestamp to the record and
the `Watermark getCurrentWatermark()` (for periodic) or the
`Watermark checkAndGetNextWatermark(T lastElement, long extractedTimestamp)` (for punctuated) is called to determine
if a new watermark should be emitted and with which timestamp.


## Kafka 生产者 (Producer)

Flink’s Kafka Producer is called `FlinkKafkaProducer08` (or `09` for Kafka 0.9.0.x versions, etc.).
It allows writing a stream of records to one or more Kafka topics.

Example:

<div class="codetabs" markdown="1">
<div data-lang="java, Kafka 0.8+" markdown="1">
{% highlight java %}
DataStream<String> stream = ...;

FlinkKafkaProducer08<String> myProducer = new FlinkKafkaProducer08<String>(
        "localhost:9092",            // broker list
        "my-topic",                  // target topic
        new SimpleStringSchema());   // serialization schema

// the following is necessary for at-least-once delivery guarantee
myProducer.setLogFailuresOnly(false);   // "false" by default
myProducer.setFlushOnCheckpoint(true);  // "false" by default

stream.addSink(myProducer);
{% endhighlight %}
</div>
<div data-lang="java, Kafka 0.10+" markdown="1">
{% highlight java %}
DataStream<String> stream = ...;

FlinkKafkaProducer010Configuration myProducerConfig = FlinkKafkaProducer010.writeToKafkaWithTimestamps(
        stream,                     // input stream
        "my-topic",                 // target topic
        new SimpleStringSchema(),   // serialization schema
        properties);                // custom configuration for KafkaProducer (including broker list)

// the following is necessary for at-least-once delivery guarantee
myProducerConfig.setLogFailuresOnly(false);   // "false" by default
myProducerConfig.setFlushOnCheckpoint(true);  // "false" by default
{% endhighlight %}
</div>
<div data-lang="scala, Kafka 0.8+" markdown="1">
{% highlight scala %}
val stream: DataStream[String] = ...

val myProducer = new FlinkKafkaProducer08[String](
        "localhost:9092",         // broker list
        "my-topic",               // target topic
        new SimpleStringSchema)   // serialization schema

// the following is necessary for at-least-once delivery guarantee
myProducer.setLogFailuresOnly(false)   // "false" by default
myProducer.setFlushOnCheckpoint(true)  // "false" by default

stream.addSink(myProducer)
{% endhighlight %}
</div>
<div data-lang="scala, Kafka 0.10+" markdown="1">
{% highlight scala %}
val stream: DataStream[String] = ...

val myProducerConfig = FlinkKafkaProducer010.writeToKafkaWithTimestamps(
        stream,                   // input stream
        "my-topic",               // target topic
        new SimpleStringSchema,   // serialization schema
        properties)               // custom configuration for KafkaProducer (including broker list)

// the following is necessary for at-least-once delivery guarantee
myProducerConfig.setLogFailuresOnly(false)   // "false" by default
myProducerConfig.setFlushOnCheckpoint(true)  // "false" by default
{% endhighlight %}
</div>
</div>

The above examples demonstrate the basic usage of creating a Flink Kafka Producer
to write streams to a single Kafka target topic. For more advanced usages, there
are other constructor variants that allow providing the following:

 * *Providing custom properties*:
 The producer allows providing a custom properties configuration for the internal `KafkaProducer`.
 Please refer to the [Apache Kafka documentation](https://kafka.apache.org/documentation.html) for
 details on how to configure Kafka Producers.
 * *Custom partitioner*: To assign records to specific
 partitions, you can provide an implementation of a `KafkaPartitioner` to the
 constructor. This partitioner will be called for each record in the stream
 to determine which exact partition the record will be sent to.
 * *Advanced serialization schema*: Similar to the consumer,
 the producer also allows using an advanced serialization schema called `KeyedSerializationSchema`,
 which allows serializing the key and value separately. It also allows to override the target topic,
 so that one producer instance can send data to multiple topics.
 
### Kafka 生产者和容错机制

With Flink's checkpointing enabled, the Flink Kafka Producer can provide
at-least-once delivery guarantees.

Besides enabling Flink's checkpointing, you should also configure the setter
methods `setLogFailuresOnly(boolean)` and `setFlushOnCheckpoint(boolean)` appropriately,
as shown in the above examples in the previous section:

 * `setLogFailuresOnly(boolean)`: enabling this will let the producer log failures only
 instead of catching and rethrowing them. This essentially accounts the record
 to have succeeded, even if it was never written to the target Kafka topic. This
 must be disabled for at-least-once.
 * `setFlushOnCheckpoint(boolean)`: with this enabled, Flink's checkpoints will wait for any
 on-the-fly records at the time of the checkpoint to be acknowledged by Kafka before
 succeeding the checkpoint. This ensures that all records before the checkpoint have
 been written to Kafka. This must be enabled for at-least-once.

**Note**: By default, the number of retries is set to "0". This means that when `setLogFailuresOnly` is set to `false`,
the producer fails immediately on errors, including leader changes. The value is set to "0" by default to avoid
duplicate messages in the target topic that are caused by retries. For most production environments with frequent broker changes,
we recommend setting the number of retries to a higher value.

**Note**: There is currently no transactional producer for Kafka, so Flink can not guarantee exactly-once delivery
into a Kafka topic.

## 使用 Kafka 时间戳和 Flink 事件时间 (在 Kafka 0.10 中)

Since Apache Kafka 0.10+, Kafka's messages can carry [timestamps](https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message), indicating
the time the event has occurred (see ["event time" in Apache Flink](../event_time.html)) or the time when the message
has been written to the Kafka broker.

The `FlinkKafkaConsumer010` will emit records with the timestamp attached, if the time characteristic in Flink is 
set to `TimeCharacteristic.EventTime` (`StreamExecutionEnvironment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)`).

The Kafka consumer does not emit watermarks. To emit watermarks, the same mechanisms as described above in 
"Kafka Consumers and Timestamp Extraction/Watermark Emission"  using the `assignTimestampsAndWatermarks` method are applicable.

There is no need to define a timestamp extractor when using the timestamps from Kafka. The `previousElementTimestamp` argument of 
the `extractTimestamp()` method contains the timestamp carried by the Kafka message.

A timestamp extractor for a Kafka consumer would look like this:
{% highlight java %}
public long extractTimestamp(Long element, long previousElementTimestamp) {
    return previousElementTimestamp;
}
{% endhighlight %}



The `FlinkKafkaProducer010` only emits the record timestamp, if `setWriteTimestampToKafka(true)` is set.

{% highlight java %}
FlinkKafkaProducer010.FlinkKafkaProducer010Configuration config = FlinkKafkaProducer010.writeToKafkaWithTimestamps(streamWithTimestamps, topic, new SimpleStringSchema(), standardProps);
config.setWriteTimestampToKafka(true);
{% endhighlight %}



## Kafka 连接器度量单位 (metrics)

Flink's Kafka connectors provide some metrics through Flink's [metrics system]({{ site.baseurl }}/monitoring/metrics.html) to analyze
the behavior of the connector.
The producers export Kafka's internal metrics through Flink's metric system for all supported versions. The consumers export 
all metrics starting from Kafka version 0.9. The Kafka documentation lists all exported metrics 
in its [documentation](http://kafka.apache.org/documentation/#selector_monitoring).

In addition to these metrics, all consumers expose the `current-offsets` and `committed-offsets` for each topic partition.
The `current-offsets` refers to the current offset in the partition. This refers to the offset of the last element that
we retrieved and emitted successfully. The `committed-offsets` is the last committed offset.

The Kafka Consumers in Flink commit the offsets back to Zookeeper (Kafka 0.8) or the Kafka brokers (Kafka 0.9+). If checkpointing
is disabled, offsets are committed periodically.
With checkpointing, the commit happens once all operators in the streaming topology have confirmed that they've created a checkpoint of their state. 
This provides users with at-least-once semantics for the offsets committed to Zookeer or the broker. For offsets checkpointed to Flink, the system 
provides exactly once guarantees.

The offsets committed to ZK or the broker can also be used to track the read progress of the Kafka consumer. The difference between
the committed offset and the most recent offset in each partition is called the *consumer lag*. If the Flink topology is consuming
the data slower from the topic than new data is added, the lag will increase and the consumer will fall behind.
For large production deployments we recommend monitoring that metric to avoid increasing latency.

## 启动 Kerberos 认证 (仅在 0.9 及以上版本)

Flink provides first-class support through the Kafka connector to authenticate to a Kafka installation
configured for Kerberos. Simply configure Flink in `flink-conf.yaml` to enable Kerberos authentication for Kafka like so:

1. Configure Kerberos credentials by setting the following -
 - `security.kerberos.login.use-ticket-cache`: By default, this is `true` and Flink will attempt to use Kerberos credentials in ticket caches managed by `kinit`. 
 Note that when using the Kafka connector in Flink jobs deployed on YARN, Kerberos authorization using ticket caches will not work. This is also the case when deploying using Mesos, as authorization using ticket cache is not supported for Mesos deployments. 
 - `security.kerberos.login.keytab` and `security.kerberos.login.principal`: To use Kerberos keytabs instead, set values for both of these properties.
 
2. Append `KafkaClient` to `security.kerberos.login.contexts`: This tells Flink to provide the configured Kerberos credentials to the Kafka login context to be used for Kafka authentication.

Once Kerberos-based Flink security is enabled, you can authenticate to Kafka with either the Flink Kafka Consumer or Producer by simply including the following two settings in the provided properties configuration that is passed to the internal Kafka client:

- Set `security.protocol` to `SASL_PLAINTEXT` (default `NONE`): The protocol used to communicate to Kafka brokers.
When using standalone Flink deployment, you can also use `SASL_SSL`; please see how to configure the Kafka client for SSL [here](https://kafka.apache.org/documentation/#security_configclients). 
- Set `sasl.kerberos.service.name` to `kafka` (default `kafka`): The value for this should match the `sasl.kerberos.service.name` used for Kafka broker configurations. A mismatch in service name between client and server configuration will cause the authentication to fail.

For more information on Flink configuration for Kerberos security, please see [here]({{ site.baseurl}}/setup/config.html).
You can also find [here]({{ site.baseurl}}/ops/security-kerberos.html) further details on how Flink internally setups Kerberos-based security.
