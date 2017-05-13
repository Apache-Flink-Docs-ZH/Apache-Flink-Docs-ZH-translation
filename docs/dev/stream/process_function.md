---
title: "过程函数 (低层次操作)"
nav-title: "过程函数"
nav-parent_id: streaming
nav-pos: 35
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

## 过程函数(ProcessFunction)

`过程函数(ProcessFunction)` 是一种低层次的流处理操作，它能访问到(无环的)流应用的基本构成单元：

  - 事件(events) (流元素)
  - 状态(state) (容错, 一致性,只在keyed stream中)
  - 定时器(timers) (事件时间和处理时间, 只在keyed stream中)

`过程函数(ProcessFunction)` 可以被认为一种提供了对有键状态(keyed state)和定时器(timers)访问的 `FlatMapFunction`。每在输入流中收到一个事件，过程函数就会被触发来对事件进行处理。

对于容错的状态(state), `过程函数(ProcessFunction)` 可以通过 `RuntimeContext` 访问Flink's [有键状态(keyed state)](state.html), 就像其它状态函数能够访问有键状态(keyed state)一样.

定时器则允许程序对处理时间和[事件时间(event time)](../event_time.html)的改变做出反应。每次对 `processElement(...)` 的调用都能拿到一个`上下文(Context)`对象,这个对象能访问到所处理元素事件时间的时间戳,还有 *定时服务器(TimerService)* 。`定时服务器(TimerService)`可以为尚未发生的处理时间或事件时间实例注册回调函数。当一个定时器到达特定的时间实例时，`onTimer(...)`方法就会被调用。在这个函数的调用期间，所有的状态(states)都会再次对应定时器被创建时key所属的states，同时被触发的回调函数也能操作这些状态。

<span class="label label-info">注意</span> 如果你希望访问有键状态(keyed state)和定时器(timers),你必须在一个键型流(keyed stream)上使用`过程函数(ProcessFunction)`:

{% highlight java %}
stream.keyBy(...).process(new MyProcessFunction())
{% endhighlight %}


## 低层的连接(Low-level Joins)

为了在两个输入源实现低层次的操作，应用可以使用 `CoProcessFunction`。该函数绑定了连个不同的输入源并且会对从两个输入源中得到的记录分别调用 `processElement1(...)` 和 `processElement2(...)` 方法。

可以按下面的步骤来实现一个低层典型的连接操作：

  - 为一个(或两个)输入源创建一个状态(state)对象
  - 在从输入源收到元素时更新这个状态(state)对象
  - 在从另一个输入源接收到元素时，扫描这个state对象并产出连接的结果

比如，你正在把顾客数据和交易数据做一个连接，并且为顾客数据保存了状态(state)。如果你担心因为事件乱序导致不能得到完整和准确的连接结果，你可以用定时器来
控制，当顾客数据的水印(watermark)时间超过了那笔交易的时间时，再进行计算和产出连接的结果。
## 例子

The following example maintains counts per key, and emits a key/count pair whenever a minute passes (in event time) without an update for that key:
下面的例子中每一个键维护了一个计数，并且会把一分钟(事件时间)内没有更新的键/值对输出:

  - 计数、键和最后一次更新时间存储在该键隐式持有的 `ValueState` 中
  - 对于每一条记录，`过程函数(ProcessFunction)` 会对这个键对应的 `ValueState` 增加计数器的值，并且调整最后一次更新时间
  - 该 `过程函数(ProcessFunction)` 也会注册一个一分钟(事件时间)后的回调函数
  - 每一次回调触发时，它会检查回调事件的时间戳和存在 `ValueState` 中最后一次更新的时间戳是否符合要求(比如，在过去的一分钟没有再发生更新)，如果符合要求则会把键/计数对传出来

<span class="label label-info">注意</span> 这个简单的列子本来可以通过会话窗口来实现。我们在这里使用 `过程函数(ProcessFunction)` 来举例说明它的基本模式。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.streaming.api.functions.ProcessFunction.Context;
import org.apache.flink.streaming.api.functions.ProcessFunction.OnTimerContext;
import org.apache.flink.util.Collector;


// 源数据流
DataStream<Tuple2<String, String>> stream = ...;

// 对一个键型流(keyed stream) 使用过程函数
DataStream<Tuple2<String, Long>> result = stream
    .keyBy(0)
    .process(new CountWithTimeoutFunction());

/**
 * 存储在state中的数据类型
 */
public class CountWithTimestamp {

    public String key;
    public long count;
    public long lastModified;
}

/**
 * 维护了计数和超时间隔的过程函数的实现
 */
public class CountWithTimeoutFunction extends ProcessFunction<Tuple2<String, String>, Tuple2<String, Long>> {

    /** 这个状态是通过过程函数来维护 */
    private ValueState<CountWithTimestamp> state;

    @Override
    public void open(Configuration parameters) throws Exception {
        state = getRuntimeContext().getState(new ValueStateDescriptor<>("myState", CountWithTimestamp.class));
    }

    @Override
    public void processElement(Tuple2<String, String> value, Context ctx, Collector<Tuple2<String, Long>> out)
            throws Exception {

        // 得到当前的计数
        CountWithTimestamp current = state.value();
        if (current == null) {
            current = new CountWithTimestamp();
            current.key = value.f0;
        }

        // 更新状态中的计数
        current.count++;

        // 设置状态中相关的时间戳
        current.lastModified = ctx.timestamp();

        // 状态回写
        state.update(current);

        // 从当前事件时间开始注册一个60s的定时器
        ctx.timerService().registerEventTimeTimer(current.lastModified + 60000);
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Tuple2<String, Long>> out)
            throws Exception {

        // 得到设置这个定时器的键对应的状态
        CountWithTimestamp result = state.value();

        // check if this is an outdated timer or the latest timer
        if (timestamp == result.lastModified + 60000) {
            // emit the state on timeout
            out.collect(new Tuple2<String, Long>(result.key, result.count));
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.common.state.ValueState
import org.apache.flink.api.common.state.ValueStateDescriptor
import org.apache.flink.streaming.api.functions.ProcessFunction
import org.apache.flink.streaming.api.functions.ProcessFunction.Context
import org.apache.flink.streaming.api.functions.ProcessFunction.OnTimerContext
import org.apache.flink.util.Collector

// 源数据流
val stream: DataStream[Tuple2[String, String]] = ...

// 对一个键型流(keyed stream) 使用过程函数
val result: DataStream[Tuple2[String, Long]] = stream
  .keyBy(0)
  .process(new CountWithTimeoutFunction())

/**
  * 存储在state中的数据类型
  */
case class CountWithTimestamp(key: String, count: Long, lastModified: Long)

/**
  * 维护了计数和超时间隔的过程函数的实现
  */
class CountWithTimeoutFunction extends ProcessFunction[(String, String), (String, Long)] {

  /** 通过过程函数来维护的状态  */
  lazy val state: ValueState[CountWithTimestamp] = getRuntimeContext
    .getState(new ValueStateDescriptor[CountWithTimestamp]("myState", classOf[CountWithTimestamp]))


  override def processElement(value: (String, String), ctx: Context, out: Collector[(String, Long)]): Unit = {
    // initialize or retrieve/update the state

    val current: CountWithTimestamp = state.value match {
      case null =>
        CountWithTimestamp(value._1, 1, ctx.timestamp)
      case CountWithTimestamp(key, count, lastModified) =>
        CountWithTimestamp(key, count + 1, ctx.timestamp)
    }

    // 状态回写
    state.update(current)

    // 从当前事件时间开始注册一个60s的定时器
    ctx.timerService.registerEventTimeTimer(current.lastModified + 60000)
  }

  override def onTimer(timestamp: Long, ctx: OnTimerContext, out: Collector[(String, Long)]): Unit = {
    state.value match {
      case CountWithTimestamp(key, count, lastModified) if (timestamp == lastModified + 60000) =>
        out.collect((key, count))
      case _ =>
    }
  }
}
{% endhighlight %}
</div>
</div>

{% top %}
