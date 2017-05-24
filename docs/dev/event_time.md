---
title: "Event Time"
nav-id: event_time
nav-show_overview: true
nav-parent_id: streaming
nav-pos: 20
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

# 事件时间 / 处理时间 / 摄取时间

Flink 支持流式程序中不同的 *时间* 概念.

- **处理时间:** 处理时间是指正在执行相关进程的机器的系统时间.

    当一个流式程序在处理时间运行时, 所有基于时间的操作 (比如时间窗口) 将会使用运行相关算子的机器的系统时间. 打个比方, 一小时运行的时间窗口将会包含在系统时间指示的整小时内到达某一特定算子的所有记录.

    处理时间是最普通的时间概念并且它不需要流和机器之间的协调.
    它提供了最好的性能和最少的延迟. 然而, 在分布式环境和异步环境中的处理时间并不提供确定性, 因为
    它容易受记录到达系统的速度(比如说从消息队列中来的), 和系统中在不同算子之间流动的记录的速度的影响.

- **事件时间:** 事件时间是生产设备中每个独立的事件发生的时间.
    这时间通常在它们进入Flink之前被嵌入记录并且 *事件时间戳* 会从记录中抽取. 一小时的事件时间窗口将包含在这小时内所有带有时间戳的记录, 而不会去管记录到达的时间, 和记录是以什么顺序到达的.

    事件时间即使是在无序事件, 延迟事件, 或者从备份或持久性日志中获得的数据中也能得出正确的结果. 在事件时间中, 时间的进度取决于数据,
    而不是挂钟上的时间. 事件时间程序必须指定如何生成 *事件时间 Watermarks*,
    也就是指示事件时间进度的机制. 下面是该机制的描述.

    由于对延迟事件和无序事件产生等待时间的本质, 事件时间的处理通常会招致一定的延迟. 正因如此, 事件时间程序都常常结合着
    *处理时间* 的操作.

- **摄取时间:** 摄取时间是事件进入Flink的时间. 在源算子中每个记录将源的当前时间作为时间戳, 并且基于时间的操作 (比如时间窗口)
    参考的就是这个时间戳.

    *摄取时间* 从概念上讲介于 *事件时间* 和 *处理时间*之间. 对比
    *处理时间*的话, 它的代价会稍微大一点, 但是会给出更多可预测的结果. 因为
    *摄取时间* 采用稳定的时间戳 (在源处分配过一次), 所以记录上不同的窗口操作将引用相同的时间戳, 而在 *处理时间* 中每个窗口算子会将记录分配给一个不同的窗口 (基于本地系统时间和任何传输延迟).

    比起 *事件时间*, *摄取时间* 程序不能处理任何无序事件或延迟数据,
    但程序无需去指定如何生成 *watermarks*.

    在内部, *摄取时间* 更像是被当成 *事件时间*, 只不过是多了时间戳自动分配和watermark 自动生成而已.

<img src="{{ site.baseurl }}/fig/times_clocks.svg" class="center" width="80%" />


### 设置一个时间特征

Filnk 数据流程序的第一个部分通常设置了基本 *时间特征*. 该设置定义了数据流的源的行为 (打个比方,它们是否会声明时间戳), 和哪个时间概念会被像 `KeyedStream.timeWindow(Time.seconds(30))`这样的窗口使用.

下面的例子展示了聚合了一小时时间窗口内的事件的某Flink程序. 窗口的行为与时间特征相适应.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val stream: DataStream[MyEvent] = env.addSource(new FlinkKafkaConsumer09[MyEvent](topic, schema, props))

stream
    .keyBy( _.getUser )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) => a.add(b) )
    .addSink(...)
{% endhighlight %}
</div>
</div>


为了在 *事件时间* 里运行此例子, 程序需要使用那些给数据直接定义事件时间并发出watermarks的源, 或者程序必须在源之后注入 *时间戳分配器 和 Watermark生成器* . 这些功能描述了如何访问时间时间戳, 以及事件流呈现的无序程度.

下面的部分描述了 *时间戳* 和 *watermarks*背后的一般机制. 为了获得如何在Flink数据流API中使用时间戳分配和assignment 生成器的信息, 请查看
[Generating Timestamps / Watermarks]({{ site.baseurl }}/dev/event_timestamps_watermarks.html).


# 事件时间和Watermarks

*注意: Flink 实现了许多从数据流模型中的技术. 为了给事件时间和watermarks有一个很好的介绍, 看一下下面的文章.*

  - [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) by Tyler Akidau
  - The [Dataflow Model paper](https://static.googleusercontent.com/media/research.google.com/en/pubs/archive/43864.pdf)


支持*事件时间* 的流处理器需要一个测量事件时间的进度的方式.
打个比方, 构建每小时时间窗口的窗口处理器在事件时间超过一小时的时候需要被通知, 以便处理器能在进程中关闭窗口.

*事件时间* 可以独立于 *处理时间* (由挂钟测量)而进行.
打个比方, 当一个程序中的事件时间和处理时间以同样的速度执行时, 某个算子的当前*事件时间* 会略微落后于*处理时间* (考虑接收事件的延迟).
另一方面, 另一个流式程序通过快速转发Kafka的topic(或另一个消息队列) 上已经缓存好的一些历史数据,
只需几秒钟就可以处理几个星期的事件时间.

------

Flink中测量事件时间的进度的机制是 **watermarks**.
Watermarks带着一个时间戳 *t* 并作为数据流的一部分流动. 一个 *Watermark(t)* 声明了流中到达时间
*t* 时的事件时间, 也就是说流中不应有更多的带有时间戳 *t' <= t* 的因素(即带有时间戳的事件大于或等于 watermark).

下面的图表展示了带有(逻辑性)时间戳, 和watermarks 内联流动的一个流. 在这个例子里事件都经过了排序
(相对于它们的时间戳而言), 也就是说 watermarks 在流中都经过了简单的周期性标记.

<img src="{{ site.baseurl }}/fig/stream_watermark_in_order.svg" alt="A data stream with events (in order) and watermarks" class="center" width="65%" />

Watermarks 对于 *乱序* 流是至关重要的, 如下图所示, 时间不按时间戳排序.
总的来说, Watermarks 是流中所有事件都到了一定时间戳的那个时间点的声明.
一旦一个 watermark 到达了某个算子, 算子会将其内部的 *事件时间时钟* 提前到水印的值.

<img src="{{ site.baseurl }}/fig/stream_watermark_out_of_order.svg" alt="A data stream with events (out of order) and watermarks" class="center" width="65%" />


## 并行流中的Watermarks 

Watermarks, 会在源功能中, 或直接在源功能之后生成. 源功能中每一个并行的子任务常常会独立生成它自己的watermarks. 这些 watermarks 定义了某特定并行源的事件时间.

watermarks 在流式程序中流动时, 它们会在到达算子时把事件时间提前. 每当一个算子提前了它的事件时间, 它就为它的继承算子生成了一个新的watermark 下线.

一些算子消耗大量的输入流; 一个联盟, 打个比方, 算子执行了一个 *keyBy(...)* 或 *partition(...)* 功能.
这样的一个算子的当前事件时间是它输入流的事件时间的最小值. 随着输入流会更新它的事件时间, 算子也会做同样的操作.

下面的图表展现了事件和watermarks在并行流中的流动,和算子追踪事件时间的一个例子.

<img src="{{ site.baseurl }}/fig/parallel_streams_watermarks.svg" alt="Parallel data streams and operators with events and watermarks" class="center" width="80%" />


## 延迟因素

某些因素破坏watermark情况的可能性是存在的, 也就是说即使在 *Watermark(t)* 已经生成之后,
更多带有时间戳 *t' <= t* 的因素将会产生. 事实上, 在许多实际情况的设置中, 某些因素会被随意的拖延, 使某个事件时间戳的所有元素要产生的时间无法被指定.
此外, 即使延迟能被约束, 拖延 过久常常是不可取的,因为它在某事件时间窗口的评测中造成了太多的延迟.

正因为这个原因, 流式程序会明确的期望一些 *延迟* 因素. 延迟因素是已超过延迟因素时间戳所指代时间的系统事件时间 (由watermarks发出信号) 之后到达的因素. 查看 [Allowed Lateness]({{ site.baseurl }}/dev/windows.html#allowed-lateness) 获取更多在事件时间窗口中如何和延迟因素一起工作的更多信息.


## 调试 Watermarks

请参考 [Debugging Windows & Event Time]({{ site.baseurl }}/monitoring/debugging_event_time.html) 部分来了解运行阶段的watermarks 调试.
