---
title: "HDFS Connector"
nav-title: Rolling File Sink
nav-parent_id: connectors
nav-pos: 5
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

此连接器提供了一个Sink，将分区文件写入Hadoop FileSystem支持的任何文件系统。要使用此连接器，请将以下依赖项添加到您的项目中：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-filesystem{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version}}</version>
</dependency>
{% endhighlight %}

请注意，Streaming连接器当前不是二进制分发的一部分。有关如何将程序与程序库打包以进行集群执行的信息，请参阅
[here]({{site.baseurl}}/dev/linking.html)。

#### Bucketing 文件 Sink

Bucketing行为以及写入操作均可被配置，我们稍后将会介绍。此处将了解如何创建一个bucketing sink，默认情况下，将会sink到按时间分割的滚动文件：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<String> input = ...;

input.addSink(new BucketingSink<String>("/base/path"));

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[String] = ...

input.addSink(new BucketingSink[String]("/base/path"))

{% endhighlight %}
</div>
</div>

唯一必需的参数是存储buckets的基本路径。可以通过指定自定义的bucketer，writer和batch大小来进一步配置sink。

默认情况下，当elements arrive时，bucketing sink将根据当前的系统时间分离，并使用日期时间格式`"yyyy-MM-dd--HH"`来命名这些buckets。此格式将传递给具有当前系统时间的`SimpleDateFormat`以形成bucket路径。每当出现新的日期时，都会创建一个新的bucket。例如，如果您有一个包含以分钟作为最细粒度的格式，您将每分钟获得一个新的bucket。每个bucket本身是一个包含几个part文件的目录：每个并行实例的sink将创建自己的part文件，当part文件变得太大时，sink将在其他part文件旁边再创建一个新的part文件。当bucket变得不活跃时，被打开的part文件将被刷新并关闭。当最近没有写入操作时，bucket将被视为不活跃。默认情况下，sink每分钟检查一次不活跃的bucket，并关闭一分钟内没有写入操作的所有bucket。可以在`BucketingSink`上使用`setInactiveBucketCheckInterval()`和`setInactiveBucketThreshold()`。

您也可以使用`BucketingSink`上的`setBucketer()`指定自定义的bucketer。 如果需要，bucketer可以使用element或tuple的属性来确定bucket目录。

默认的writer是`StringWriter`。这将在传入的elements上调用`toString()`，并将它们写入part文件，用换行符分隔。 要在`BucketingSink`上指定一个自定义的writer，请使用`setWriter()`。如果要编写Hadoop SequenceFiles，可以使用系统提供的`SequenceFileWriter`其可被配置为使用compression。

最后一个配置选项是批量大小。 这指定何时应该关闭part文件并启动一个新的part文件。 （默认part文件大小为384 MB）。

示例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<IntWritable,Text>> input = ...;

BucketingSink<String> sink = new BucketingSink<String>("/base/path");
sink.setBucketer(new DateTimeBucketer<String>("yyyy-MM-dd--HHmm"));
sink.setWriter(new SequenceFileWriter<IntWritable, Text>());
sink.setBatchSize(1024 * 1024 * 400); // this is 400 MB,

input.addSink(sink);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[Tuple2[IntWritable, Text]] = ...

val sink = new BucketingSink[String]("/base/path")
sink.setBucketer(new DateTimeBucketer[String]("yyyy-MM-dd--HHmm"))
sink.setWriter(new SequenceFileWriter[IntWritable, Text]())
sink.setBatchSize(1024 * 1024 * 400) // this is 400 MB,

input.addSink(sink)

{% endhighlight %}
</div>
</div>

将创建一个写入到遵循该schema的bucket文件的sink：

```
/base/path/{date-time}/part-{parallel-task}-{count}
```

其中`date-time`是从日期/时间格式获取的字符串，`parallel-task`是并行sink实例的索引，`count`是根据批量大小而创建的part文件的运行数。

更多详细信息，请参阅 JavaDoc for
[BucketingSink](http://flink.apache.org/docs/latest/api/java/org/apache/flink/streaming/connectors/fs/bucketing/BucketingSink.html).
