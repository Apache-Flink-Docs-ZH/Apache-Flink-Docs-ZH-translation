---
title: "预定义的 Timestamp Extractors / Watermark Emitters"
nav-parent_id: event_time
nav-pos: 2
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

正如在上一节[ timestamp 与 watermark 处理]({{ site.baseurl }}/dev/event_timestamps_watermarks.html)中提到的，Flink 提供了抽象类来让开发者给自己的 timestamps（时间戳）并发送他们自己的watermark（水印）赋值。更确切地说，开发者需要依照不同的使用场景来实现接口 `AssignerWithPeriodicWatermarks` 或接口 `AssignerWithPunctuatedWatermarks`。简而言之，前一个接口将会周期性发送 watermark，而第二个接口则根据到达数据的一些属性发送 watermark，例如:一旦在流中碰到一个特殊的元素, 便发送 watermark。

为了进一步简化开发者开发类似的任务，Flink自带了一些预先实现的时间戳分配器（timestamp assigners）。本节列举了关于这些时间戳分配器的相关内容。除了开箱即用的函数，这些预先实现的分配器还可以作为自定义分配器的示例。


### **递增时间戳的分配器**


*周期性*的 watermark 生成有一种最简单的特殊情况--给定源任务锁定的 timestamp 呈升序状态。在这种情况下，由于没有更早的 timestamp 出现，当前 timestamp 可以一直扮演着 watermark 的角色。

请注意：上述的情况只有在 timestamp 是递增的*并行数据源任务*时才是必要的。例如：在一个特定设置中，一个 Kafka 分区是由一个并行数据源实例读取的，那么 timestamp 只需要在每个 Kafka 分区中增加即可。每当并行流被关闭、统一、链接或是合并的时候，Flink的 watermark 合并机制都会产生正确的 watermark。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyEvent>() {

        @Override
        public long extractAscendingTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignAscendingTimestamps( _.getCreationTime )
{% endhighlight %}
</div>
</div>

### **允许固定数量延迟的分配器**

周期性 watermark 产生的另外一种情况是在当 watermark 滞后于最大值（即事件时间）时，timestamp 会被一段固定时间流锁定。这种方法包括了在流中遇到的最大延迟，且这个最大延迟可以被提前知道的情况。例如：当在创建元素包含 timestamp 的自定义源时，这些 timestamp 只可在固定时间的测试中进行散播。对于这些情况，FLink提供了`BoundedOutOfOrdernessTimestampExtractor` 作为 `maxOutOfOrderness` 的一个参数。即在一个 element（元素）被给定窗口，在计算最终结果忽略之前（即该element过期前），所允许该 element 迟到的最大 lateness（延迟）。lateness 与 `t-t_w` （t减t_w,译者注）相对应，其中 `t` 指代元素的 timestamp (event-time) ，而 `t_w` 则指代先前的 watermark。如果 `lateness>0`，则可认为此时该 element 延迟，而且在默认情况下，当计算相应窗口结果时，该 element 会被忽略掉。想了解更多关于使用延迟元素的知识，请参阅 [allowed lateness]({{ site.baseurl }}/dev/windows.html#allowed-lateness)的相关文档。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<MyEvent>(Time.seconds(10)) {

        @Override
        public long extractTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[MyEvent] = ...

val withTimestampsAndWatermarks = stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[MyEvent](Time.seconds(10))( _.getCreationTime ))
{% endhighlight %}
</div>
</div>
