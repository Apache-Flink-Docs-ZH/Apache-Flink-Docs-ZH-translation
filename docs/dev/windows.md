---
title: "Windows"
nav-parent_id: streaming
nav-id: windows
nav-pos: 10
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

Windows是处理无限流的核心。 Windows将流分隔成有限大小的“桶”，以供我们进行计算。 本文档重点介绍Flink中Windows如何工作，以及程序员如何充分利用其提供的功能。

Flink Window编程的一般结构如下。 第一个片段是*keyed*流，而第二个是*non-keyed*流。可以看出，唯一的区别是在*keyed*流中执行的`keyBy(...)`和`window(...)`在*non-keyed*流中被换成`windowAll(...)`。 这些也将在本页的剩余部分进行说明。

**Keyed Windows**

    stream
           .keyBy(...)          <-  keyed 和 non-keyed windows的区别
           .window(...)         <-  必选: "assigner"
          [.trigger(...)]       <-  可选: "trigger" (默认 default trigger)
          [.evictor(...)]       <-  可选: "evictor" (默认无 evictor)
          [.allowedLateness()]  <-  可选, 默认值为0
           .reduce/fold/apply() <-  必选: "function"

**Non-Keyed Windows**

    stream
           .windowAll(...)      <-  必选: "assigner"
          [.trigger(...)]       <-  可选: "trigger" (默认 default trigger)
          [.evictor(...)]       <-  可选: "evictor" (默认无 evictor)
          [.allowedLateness()]  <-  可选, 默认值为0
           .reduce/fold/apply() <-  必选: "function"

在上面，方括号（[...]）中的命令是可选的。这说明Flink允许你以多种方式自定义你的window逻辑，以满足你的需求。

* This will be replaced by the TOC
{:toc}

## Window 生命周期

简单的说，一个window在属于此window的第一个元素到达时创建，window完全删除的条件是：时间（事件或处理时间）达到该window的结束时间戳，并加上用户指定的允许的延迟，窗口被完全删除(参见 [Allowed Lateness](#allowed-lateness))。Flink保证仅对基于时间的window进行删除，而不适用于其他类型的窗口，比如全局window(参见 [Window Assigner](#window-assigners))。例如，使用基于事件时间的window策略，每5分钟创建不重叠（或翻滚tumbling）的window，并且允许的延迟时间为1分钟，则Flink会在时间戳落在`12:00`和`12:05`的第一个元素到达时创建一个新window，当watermark通过`12:06`的时间戳时删除该window。

此外，每个window都有一个`Trigger` (参见 [Triggers](#triggers))和一个附着在trigger上的functon(`WindowFunction`，`ReduceFunction` 或者
`FoldFunction`) (参见 [Window Functions](#window-functions)) 。这个function包含了将要对window里包含的内容的计算逻辑，而`Trigger`指明了window可以用于function计算的条件。trigger的策略可能是“当window中的元素个数大于4时”，或者“当watermark到达window的末尾时”。一个trigger还可以决定在window的生命周期内的任意时刻清除该window的内容。在这种情况下，清除仅指清除window中的元素，而*不是*window元数据。这意味着新数据仍然可以添加到该window。

除上述之外，你还可以指定一个`Evictor`(参见 [Evictors](#evictors))，它将在trigger触发之后以及在应用function逻辑之前和/或之后从window中移除元素。

以下我们将详细介绍上述各个组件。我们从上面的代码片段开始(参见 [Keyed vs Non-Keyed Windows](#keyed-vs-non-keyed-windows)，[Window Assigner](#window-assigner)和[Window Function](#window-function))，然后再介绍可选的部分。

## Keyed vs Non-Keyed Windows

第一件要指定的事情是你的stream是否需要按key拆分。这必须在定义window之前完成。使用`keyBy(...)`将会把你的无限stream拆分为逻辑上keyed的streams。如果没有调用`keyBy(...)`，你的stream就不是keyed。

如果是keyed streams，则进入的事件的任意属性都可以用来作为key（更多细节参见 [这里]({{ site.baseurl }}/dev/api_concepts.html#specifying-keys)）。keyed stream也将允许你的window计算并行的在多个task上进行，因为每个逻辑keyed stream和其他keyed stream是完全独立的。所有key相同的元素都会被发送到同一个并行的task。

如果是non-keyed streams，则原始stream将不会被拆分为多个逻辑上的streams，并且所有的windowing计算逻辑都会值被一个task执行，即并行度为1.

## Window Assigners

指定了流是否需要按key拆分后，下一步是定义*window assigner*。window assigner定义元素如何分配给window。对*keyed* streams，通过`window(...)`方法指定`WindowAssigner`。对*non-keyed* streams，通过`windowAll()`方法指定`WindowAssigner`。

一个`WindowAssigner`负责对每个进入的元素分配一个或者多个window。Flink自带了针对用户常见场景的window assigners，分别是*tumbling windows*，
*sliding windows*，*session windows*和*global windows*。你也可以继承`WindowAssigner`类来实现自定义window assigner。所有内置的window assigners (除了全局windows) 都是基于时间来给窗口分配元素，这里的是回见可以是processing time，也可以是event
time。请参见[event time]({{ site.baseurl }}/dev/event_time.html) 了解processing time和event time的区别以及时间戳和watermarks如何生成。

接下来，我们将展示Flink预定义的window assigners的工作原理以及如何在DataStream程序中使用它们。以下图形可视化每个assigner的运行过程。紫色圆圈表示stream中的元素，它们根据key进行了分区（例子中key是*user 1*，*user 2*和*user 3*）。x轴表示时间进度。

### Tumbling Windows

一个*tumbling windows* assigner分配每个元素到一个具有自定*window size*的window。Tumbling windows有固定size并且不重叠。例如，如果你指定一个tumbling
window的size为5分钟，则在当前window被计算后，每五分钟将创建一个新window，如下图所示。

<img src="{{ site.baseurl }}/fig/tumbling-windows.svg" class="center" style="width: 100%;" />

如下代码片段示意了如何使用tumbling windows。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>);

// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// tumbling event-time windows
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)

// tumbling processing-time windows
input
    .keyBy(<key selector>)
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .<windowed transformation>(<window function>)

// daily tumbling event-time windows offset by -8 hours.
input
    .keyBy(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.days(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

时间间隔可以通过使用`Time.milliseconds(x)`，`Time.seconds(x)`，`Time.minutes(x)`中的任意一个来指定。


如最后一个例子所示，tumbling window assigner还支持可选的`offset`参数，用于设置window的对齐位置。例如，window size为1小时的tumbling windows如果没有设置offset，则默认对齐位置为每个整点小时，那么你将得到像`1:00:00.000 - 1:59:59.999`，`2:00:00.000 - 2:59:59.999`等等这样的window。你可以通过设置offset来改变对齐位置。如果设置offset为15分钟，那么你得到像`1:15:00.000 - 2:14:59.999`, `2:15:00.000 - 3:14:59.999`等等这样的window。设置offset的一个重要场景是对window调整时区（默认时区为UTC-0）。例如，在中国你一般会指定offset为`Time.hours(-8)`。

### Sliding Windows

*sliding windows* assigner分配元素到具有固定length的window。和tumbling
windows assigner类似，window的size通过*window size*参数指定，另外通过*window slide*参数控制sliding window的新建频率。因此当slide比window size小的时候多个sliding windows会重叠，此时元素会被分配给多个windows。

例如，你可以配置size为10分钟slide为5分钟的sliding window。此时，每过5分钟会新建一个包含过去10分钟内到达事件的window，如下图所示。

<img src="{{ site.baseurl }}/fig/sliding-windows.svg" class="center" style="width: 100%;" />

如下代码片段示意了如何使用sliding windows。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>);

// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// sliding event-time windows
input
    .keyBy(<key selector>)
    .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)

// sliding processing-time windows
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.seconds(10), Time.seconds(5)))
    .<windowed transformation>(<window function>)

// sliding processing-time windows offset by -8 hours
input
    .keyBy(<key selector>)
    .window(SlidingProcessingTimeWindows.of(Time.hours(12), Time.hours(1), Time.hours(-8)))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

时间间隔可以通过使用`Time.milliseconds(x)`，`Time.seconds(x)`，`Time.minutes(x)`中的任意一个来指定。

如最后一个例子所示，sliding window assigner也支持可选的`offset`参数，用于设置window的对齐位置。例如，window size为1小时slide为30分钟的sliding windows如果没有设置offset，则默认对齐位置为每个整点小时，那么你将得到像`1:00:00.000 - 1:59:59.999`, `1:30:00.000 - 2:29:59.999`等等这样的window。你可以通过设置offset来改变对齐位置。如果设置offset为15分钟，那么你得到像`1:15:00.000 - 2:14:59.999`, `1:45:00.000 - 2:44:59.999`等等这样的window。设置offset的一个重要场景是对window调整时区（默认时区为UTC-0）。例如，在中国你一般会指定offset为`Time.hours(-8)`。

### Session Windows

*session windows* assigner通过session的活跃度分组元素。不同于*tumbling windows*和*sliding windows*，Session windows不重叠并且没有固定的起止时间。一个session window在一段时间内没有接收到元素时，即当出现非活跃间隙时关闭。一个session window assigner通过配置*session gap*来指定非活跃周期的时长。当超过这个时长，当前session关闭，后续元素被分配到新的session window。

<img src="{{ site.baseurl }}/fig/session-windows.svg" class="center" style="width: 100%;" />

如下代码片段示意了如何使用session windows。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

// event-time session windows
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);

// processing-time session windows
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

// event-time session windows
input
    .keyBy(<key selector>)
    .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)

// processing-time session windows
input
    .keyBy(<key selector>)
    .window(ProcessingTimeSessionWindows.withGap(Time.minutes(10)))
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

时间间隔可以通过使用`Time.milliseconds(x)`，`Time.seconds(x)`，`Time.minutes(x)`中的任意一个来指定。

<span class="label label-danger">注意</span> 由于session windows没有固定起止时间，所以它们的处理方式不同于tumbling和sliding windows。在内部，一个session window operator对每个到达的记录创建一个新window，并且当这些windows的距离比定义的间隙更近则合并这些windows。为了可以进行合并，一个session window operator需要一个合并的[Trigger](#triggers)和一个合并的[Window Function](#window-functions)，比如`ReduceFunction`或者`WindowFunction`
(`FoldFunction`不能合并)

### Global Windows

*global windows* assigner把具有相同key的所有元素分配给相同的单个*全局window*。这种window语义仅当你同时制定一个自定义的[trigger](#triggers)的时候才有意义。否则，不会执行计算，因为global window不知道聚合元素何时到结尾。

<img src="{{ site.baseurl }}/fig/non-windowed.svg" class="center" style="width: 100%;" />

如下代码片段示意了如何使用global window。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

input
    .keyBy(<key selector>)
    .window(GlobalWindows.create())
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

## Window Functions

定义window assigner之后，我们需要在窗口上指定要执行的计算逻辑。*window function*的职责是一旦系统确定window准备就绪（参见 [triggers](#triggers)了解Flink如何确定window是否就绪），就处理每个（可能是根据key拆分的）window的元素。

window function可以是`ReduceFunction`，`FoldFunction`或者`WindowFunction`三者之一。前两个可以更有效地执行（参见 [State Size](#state size)部分），因为Flink可以增量地聚合每个到达window的元素。`WindowFunction`获取包含在window中的所有元素的`Iterable`以及元素所属window的其他元信息。

使用`WindowFunction`的window transformation不如其他情况高效，因为Flink必须在调用函数之前在内部缓冲window中的*所有*元素。这可以通过将`WindowFunction`与`ReduceFunction`或`FoldFunction`相结合来进行缓解，以获得window元素的增量聚合和`WindowFunction`接收到的其他window元数据。我们将看看每个这些示例的变体。

### ReduceFunction

`ReduceFunction`指定输入的两个元素如何组合以产生相同类型的输出元素。Flink使用`ReduceFunction`增量地聚合window的元素。

一个`ReduceFunction`的定义和使用如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce(new ReduceFunction<Tuple2<String, Long>> {
      public Tuple2<String, Long> reduce(Tuple2<String, Long> v1, Tuple2<String, Long> v2) {
        return new Tuple2<>(v1.f0, v1.f1 + v2.f1);
      }
    });
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .reduce { (v1, v2) => (v1._1, v1._2 + v2._2) }
{% endhighlight %}
</div>
</div>

上述示例计算了window中所有元组类型的元素中第二个字段的总和。

### FoldFunction

`FoldFunction`指定window的一个输入元素如何与一个同类型的输出元素结合。对于添加到窗口的每个元素和当前输出值，`FoldFunction`被不断地调用。第一个元素与输出类型的预定义初始值组合。

一个`FoldFunction`的定义和使用如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("", new FoldFunction<Tuple2<String, Long>, String>> {
       public String fold(String acc, Tuple2<String, Long> value) {
         return acc + value.f1;
       }
    });
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .fold("") { (acc, v) => acc + v._2 }
{% endhighlight %}
</div>
</div>

上述示例将所有输入的`Long`值追加到初始值为空的`String`。

<span class="label label-danger">注意</span> `fold()`不能用于session windows以及其他可合并的windows。

### WindowFunction - 通用场景

`WindowFunction`获取包含window所有元素的`Iterable`，并提供最灵活的window functions。不过这是以性能和资源消耗为代价的，因为元素不能增量地聚合，而要在内部缓冲，直到window就绪才能进行处理。

一个`WindowFunction`的签名如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function, Serializable {

  /**
   * Evaluates the window and outputs none or several elements.
   *
   * @param key The key for which this window is evaluated.
   * @param window The window that is being evaluated.
   * @param input The elements in the window being evaluated.
   * @param out A collector for emitting elements.
   *
   * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
   */
  void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
trait WindowFunction[IN, OUT, KEY, W <: Window] extends Function with Serializable {

  /**
    * Evaluates the window and outputs none or several elements.
    *
    * @param key    The key for which this window is evaluated.
    * @param window The window that is being evaluated.
    * @param input  The elements in the window being evaluated.
    * @param out    A collector for emitting elements.
    * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  def apply(key: KEY, window: W, input: Iterable[IN], out: Collector[OUT])
}
{% endhighlight %}
</div>
</div>

一个`WindowFunction`的定义和使用如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction());

/* ... */

public class MyWindowFunction implements WindowFunction<Tuple<String, Long>, String, String, TimeWindow> {

  void apply(String key, TimeWindow window, Iterable<Tuple<String, Long>> input, Collector<String> out) {
    long count = 0;
    for (Tuple<String, Long> in: input) {
      count++;
    }
    out.collect("Window: " + window + "count: " + count);
  }
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .apply(new MyWindowFunction())

/* ... */

class MyWindowFunction extends WindowFunction[(String, Long), String, String, TimeWindow] {

  def apply(key: String, window: TimeWindow, input: Iterable[(String, Long)], out: Collector[String]): () = {
    var count = 0L
    for (in <- input) {
      count = count + 1
    }
    out.collect(s"Window $window count: $count")
  }
}
{% endhighlight %}
</div>
</div>

该例子展示了一个用于计数元素个数的`WindowFunction`。此外，这个window function把关于window的信息添加到输出。

<span class="label label-danger">注意</span> 使用`WindowFunction`来实现简单的聚合（如计数）是非常低效的。下一节将介绍如何将`ReduceFunction`与`WindowFunction`组合以实现增量聚合并获得添加到`WindowFunction`的信息。

### ProcessWindowFunction

在`WindowFunction`可以使用的地方，你也可以使用`ProcessWindowFunction`。它与`WindowFunction`十分相似，只是它的接口允许查询更多关于将要进行计算的window的上下文信息。

`ProcessWindowFunction`的接口如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> implements Function {

    /**
     * Evaluates the window and outputs none or several elements.
     *
     * @param key The key for which this window is evaluated.
     * @param context The context in which the window is being evaluated.
     * @param elements The elements in the window being evaluated.
     * @param out A collector for emitting elements.
     *
     * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
     */
    public abstract void process(
            KEY key,
            Context context,
            Iterable<IN> elements,
            Collector<OUT> out) throws Exception;

    /**
     * The context holding window metadata
     */
    public abstract class Context {
        /**
         * @return The window that is being evaluated.
         */
        public abstract W window();
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
abstract class ProcessWindowFunction[IN, OUT, KEY, W <: Window] extends Function {

  /**
    * Evaluates the window and outputs none or several elements.
    *
    * @param key      The key for which this window is evaluated.
    * @param context  The context in which the window is being evaluated.
    * @param elements The elements in the window being evaluated.
    * @param out      A collector for emitting elements.
    * @throws Exception The function may throw exceptions to fail the program and trigger recovery.
    */
  @throws[Exception]
  def process(
      key: KEY,
      context: Context,
      elements: Iterable[IN],
      out: Collector[OUT])

  /**
    * The context holding window metadata
    */
  abstract class Context {
    /**
      * @return The window that is being evaluated.
      */
    def window: W
  }
}
{% endhighlight %}
</div>
</div>

用法示意：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, Long>> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .process(new MyProcessWindowFunction());
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(String, Long)] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .process(new MyProcessWindowFunction())
{% endhighlight %}
</div>
</div>

### WindowFunction与增量聚合结合

`WindowFunction`可以与`ReduceFunction`或`FoldFunction`组合，以便在元素到达window时增量地聚合元素。当window关闭时，`WindowFunction`将得到聚合结果。这样就允许在访问`WindowFunction`的其他的window元信息的同时增量对window进行计算。

<span class="label label-info">注意</span> 你也可以使用`ProcessWindowFunction`取代`WindowFunction`来对window进行增量聚合。

#### 使用FoldFunction对window进行增量聚合

以下示例展示了增量`FoldFunction`如何与`WindowFunction`组合以获取window中的事件数，并返回window的key和结束时间。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .fold(new Tuple3<String, Long, Integer>("",0L, 0), new MyFoldFunction(), new MyWindowFunction())

// Function definitions

private static class MyFoldFunction
    implements FoldFunction<SensorReading, Tuple3<String, Long, Integer> > {

  public Tuple3<String, Long, Integer> fold(Tuple3<String, Long, Integer> acc, SensorReading s) {
      Integer cur = acc.getField(2);
      acc.setField(2, cur + 1);
      return acc;
  }
}

private static class MyWindowFunction
    implements WindowFunction<Tuple3<String, Long, Integer>, Tuple3<String, Long, Integer>, String, TimeWindow> {

  public void apply(String key,
                    TimeWindow window,
                    Iterable<Tuple3<String, Long, Integer>> counts,
                    Collector<Tuple3<String, Long, Integer>> out) {
    Integer count = counts.iterator().next().getField(2);
    out.collect(new Tuple3<String, Long, Integer>(key, window.getEnd(),count));
  }
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[SensorReading] = ...

input
 .keyBy(<key selector>)
 .timeWindow(<window assigner>)
 .fold (
    ("", 0L, 0),
    (acc: (String, Long, Int), r: SensorReading) => { ("", 0L, acc._3 + 1) },
    ( key: String,
      window: TimeWindow,
      counts: Iterable[(String, Long, Int)],
      out: Collector[(String, Long, Int)] ) =>
      {
        val count = counts.iterator.next()
        out.collect((key, window.getEnd, count._3))
      }
  )

{% endhighlight %}
</div>
</div>

#### 使用ReduceFunction对window进行增量聚合

以下示例展示了增量`ReduceFunction`如何与`WindowFunction`组合以返回window中的最小事件以及window的开始时间。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<SensorReading> input = ...;

input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .reduce(new MyReduceFunction(), new MyWindowFunction());

// Function definitions

private static class MyReduceFunction implements ReduceFunction<SensorReading> {

  public SensorReading reduce(SensorReading r1, SensorReading r2) {
      return r1.value() > r2.value() ? r2 : r1;
  }
}

private static class MyWindowFunction
    implements WindowFunction<SensorReading, Tuple2<Long, SensorReading>, String, TimeWindow> {

  public void apply(String key,
                    TimeWindow window,
                    Iterable<SensorReading> minReadings,
                    Collector<Tuple2<Long, SensorReading>> out) {
      SensorReading min = minReadings.iterator().next();
      out.collect(new Tuple2<Long, SensorReading>(window.getStart(), min));
  }
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val input: DataStream[SensorReading] = ...

input
  .keyBy(<key selector>)
  .timeWindow(<window assigner>)
  .reduce(
    (r1: SensorReading, r2: SensorReading) => { if (r1.value > r2.value) r2 else r1 },
    ( key: String,
      window: TimeWindow,
      minReadings: Iterable[SensorReading],
      out: Collector[(Long, SensorReading)] ) =>
      {
        val min = minReadings.iterator.next()
        out.collect((window.getStart, min))
      }
  )

{% endhighlight %}
</div>
</div>

## Triggers

`Trigger`定义了window（通过*window assigner*形成的）何时准备好被*window function*处理。每个`WindowAssigner`默认都有一个`Trigger`。如果默认的trigger不符合你的需求，你可以使用`trigger(...)`自定义trigger。

trigger的接口有5种方法允许`Trigger`对不同事件作出反应：

* `onElement()`方法在每个元素被添加到window的时候被调用。
* `onEventTime()`方法在注册的event-time的timer触发的时候被调用。
* `onProcessingTime()`方法在注册的processing-time的timer触发的时候被调用。
* `onMerge()`方法与有状态的triggers相关，并且在相应的window合并时合并两个trigger的状态，比如再使用session windows的时候。
* 最后`clear()`方法执行删除相应window所需的任何操作。

以上方法有两件事要注意：

1) 前3个方法通过返回一个`TriggerResult`来决定如何对当前元素作出反应，可能的反应如下：

* `CONTINUE`：什么也不做，
* `FIRE`：触发计算，
* `PURGE`：清除window中的元素，
* `FIRE_AND_PURGE`：触发计算，然后清除window中的元素。

2) 以上方法都可以注册processing或者event-time的计时器(timer)以便后续使用。

### 触发和清除

一旦trigger确定一个window已准备好进行处理，该trigger将触发，即它返回`FIRE`或`FIRE_AND_PURGE`。这是给window operator的信号以让发出window的计算结果。如果一个window使用`WindowFunction`，则所有的元素都被传递给该WindowFunction（可能在将元素传递给evictor之后）。如果一个window使用`FoldFunction`的`ReduceFunction`，则只会发出他们所需的聚合结果。

当一个trigger触发，它可以是`FIRE`或者`FIRE_AND_PURGE`。二者区别是`FIRE`保留window的内容，而`FIRE_AND_PURGE`清空内容。默认情况下，内置预实现的triggers只是`FIRE`而不是清除window的状态。

<span class="label label-danger">注意</span> Purge将会清除window的内容，并保留关于window的所有潜在相关的元信息和trigger状态。

### WindowAssigners的默认Triggers

`WindowAssigner`默认的`Trigger`适用于多种场景。例如，所有event-time的window assigners都有一个`EventTimeTrigger`座位默认trigger。该trigger在watermark通过window末尾时触发。

<span class="label label-danger">注意</span> `GlobalWindow`默认的trigger是`NeverTrigger`，该trigger从不触发。所以在使用`GlobalWindow`的时候你必须自定义trigger。

<span class="label label-danger">注意</span> 通过`trigger()`方法，你可以覆盖`WindowAssigner`的默认trigger。例如，如果你为`TumblingEventTimeWindows`指定了`CountTrigger`，则窗口触发不再根据时间的进度，而是通过计数。当前，如果你想同时基于时间进度和计数触发window，你需要自定义trigger。

### 内置Triggers和自定义Triggers

Flink comes with a few built-in triggers.

* `EventTimeTrigger`（上文已提及）基于watermarks推进的event-time进度来触发。
* `ProcessingTimeTrigger`基于processing time触发。
* `CountTrigger` 当window中的元素数量超过给定的限制就触发。
* `PurgingTrigger`把别的trigger作为参数，并将别的trigger转换为purging trigger。

如果你需要实现一个自定义trigger，你应该查看抽象类{% gh_link /flink-streaming-java/src/main/java/org/apache/flink/streaming/api/windowing/triggers/Trigger.java "Trigger" %}。请注意，该API仍在演化过程中，在未来的Flink版本中可能发生变化。

## Evictors

Flink的window模型允许对`WindowAssigner`和`Trigger`指定可选的`Evictor`。这可以通过`evictor(...)`方法来完成（如本文档首部所示）。 evictor能够在trigger触发*之后*以及应用window function*之前和/或之后*从window中移除元素。为此，`Evictor`接口有2个方法：

    /**
     * Optionally evicts elements. Called before windowing function.
     *
     * @param elements The elements currently in the pane.
     * @param size The current number of elements in the pane.
     * @param window The {@link Window}
     * @param evictorContext The context for the Evictor
     */
    void evictBefore(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);

    /**
     * Optionally evicts elements. Called after windowing function.
     *
     * @param elements The elements currently in the pane.
     * @param size The current number of elements in the pane.
     * @param window The {@link Window}
     * @param evictorContext The context for the Evictor
     */
    void evictAfter(Iterable<TimestampedValue<T>> elements, int size, W window, EvictorContext evictorContext);

`evictBefore()`包含了应用window function之前的驱逐逻辑，而`evictAfter()`包含了应用window function之后的驱逐逻辑，当然在应用window function之前被逐出的元素将不被`evictAfter()`处理。

Flink自带了3种预定义的evictors。它们是：

* `CountEvictor`：保留window中用户指定数量的元素数量，并从window的头部丢弃剩余的元素。
* `DeltaEvictor`：通过`DeltaFunction`和一个`threshold`计算window缓冲区中最后一个元素与剩余的最后一个元素之间的差值，并删除差值大于或者等于threshold的元素。
* `TimeEvictor`：通过毫秒为单位的参数`interval`，对给定的window找到其中元素时间戳的最大值`max_ts`，并删除时间戳小于`max_ts - interval`的元素。

<span class="label label-info">默认</span> 默认情况下，所有内置的evictors都在window function之前应用其逻辑。

<span class="label label-danger">注意</span> 指定evictor会阻止一切预聚合，因为window的所有元素都必须在应用计算逻辑前先传给evictor进行处理。

<span class="label label-danger">注意</span> Flink不保证window内元素的顺序。这意味着虽然evictor从window的头部开始驱逐元素，但是并不代表这些头部元素一定是早到或者晚到window的。


## Allowed Lateness

当使用*event-time* window时，可能元素会晚到，即Flink用于跟踪event-time进度的watermark已经达超过了window的结束时间戳。参见[event time](./event_time.html)以及[late elements](./event_time.html#late-elements)了解更多关于Flink如何处理event time。


默认情况下，当watermark超过window的末尾时，晚到的元素会被丢弃。但是Flink也允许为window operator指定最大*allowed lateness*。*allowed lateness*表示在彻底删除元素之前最多可以容忍多长时间晚到的元素，其默认值为0。元素如果在*allowed lateness*通过window末尾之后但在window结束时间加上*allowed lateness*之前到达，仍会被添加到window。在用某些trigger时，晚到但未被丢弃的元素可能会再次触发window。`EventTimeTrigger`就是这种trigger。

为了支持该功能，Flink会保持window的状态，直到*allowed lateness*到期。一旦到期，Flink会删除window并删除其状态，如[Window Lifecycle](#window-lifecycle)部分所述。

<span class="label label-info">默认</span> 默认情况下，*allowed lateness*值为`0`。也就是说晚于watermark到达的元素将被丢弃。

你可以像下面这样设置allowed lateness：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<T> input = ...;

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[T] = ...

input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .<windowed transformation>(<window function>)
{% endhighlight %}
</div>
</div>

<span class="label label-info">注意</span> 当使用`GlobalWindows`的window assigner时，不会有元素被认为是晚到的，因为global window的结束时间是`Long.MAX_VALUE`。

### 把晚到元素当做side output

使用Flink的[side output]({{ site.baseurl }}/dev/stream/side_output.html)功能时，你可以获取到因为晚到被丢弃的元素流。

首先你需要在windowed流上通过`sideOutputLateData(OutputTag)`指明你想要获取晚到的元素，然后你就能在windowed operation的结果中获取到side-output流：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final OutputTag<T> lateOutputTag = new OutputTag<T>("late-data"){};

DataStream<T> input = ...;

DataStream<T> result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>);

DataStream<T> lateStream = result.getSideOutput(lateOutputTag);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val lateOutputTag = OutputTag[T]("late-data")

val input: DataStream[T] = ...

val result = input
    .keyBy(<key selector>)
    .window(<window assigner>)
    .allowedLateness(<time>)
    .sideOutputLateData(lateOutputTag)
    .<windowed transformation>(<window function>)

val lateStream = result.getSideOutput(lateOutputTag)
{% endhighlight %}
</div>
</div>

### Late elements的考虑

当指定allowed lateness大于0，在watermark通过window结尾时，window的内容仍需要保留。此时，当一个晚到但不该被丢弃的元素到达时，它可能会导致window的另一次触发。这些触发被称为`late firings`，因为是由晚到的事件所导致的。而`main firing`是指window的第一次触发。在使用session windows时，late firings可能进一步导致windows的合并因为它们可能"弥合"了两个此前已经存在的但是未被合并的window。

<span class="label label-info">注意</span> 你应该注意到，通过late firing发出的元素应该被当做先前计算的修正，也就是说你的数据流将会包含相同计算的多个结果。根据你应用的需要，你可能需要考虑这些重复计算结果或者对它们进行去重处理。

## 有用状态大小的考虑

Windows的时间跨度可以被定义得很大（比如数天，数周或者数月），当然这会累积非常大的状态量。估计window的存储需求时要注意如下几个规则：

1. Flink对元素归于某个window时对元素创建一个副本。所以，tumbling windows持有每个元素的一个副本（一个元素只能属于一个window，除非它后来被删除），而sliding windows会为每个元素创建若干个副本，如[Window Assigners](#window-assigners) 章节中所描述。因此一个size为1天slide为1秒的sliding window of可能不是个好注意。

2. `FoldFunction`和`ReduceFunction`可以显着降低存储要求，因为它们会聚合元素，并且每个window只存储一个值。相比之下，只有当必须累积全部元素才能计算时才使用`WindowFunction`。

3. 使用`Evictor`可以防止预聚合，因为window的所有元素都必须在应用计算逻辑前先传给evictor进行处理(参见 [Evictors](#evictors))。
