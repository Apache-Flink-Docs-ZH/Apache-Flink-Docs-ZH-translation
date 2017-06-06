---
title: "Working with State"
nav-parent_id: streaming
nav-pos: 40
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

* ToC
{:toc}

带有状态的（Stateful）函数与算子可以在处理单个元素/事件的时候存储数据，使“状态（state）"成为任何类型的复杂操作的的关键组成部分。例如：

  - 当我们需要建立一个搜索固定事件模式(events patterns)的应用时，状态（state）将可以存储至今为止所有事件。
  - 当需要每分钟聚合一次事件（events）的时候，状态(state)将会hold住那些待处理的聚合。
  - 当为数据流训练一个机器学习模型的时候，状态(state)会存储当前时刻模型的所有参数。

为了使状态（state）具有容错性，Flink需要知道状态(state)并对他做[检查点(checkpoint)](checkpointing.html)操作。
在许多情况下，Flink还支持在应用中*管理*状态(state),这就意味着Flink需要用到内存管理（或者内存不够时写入磁盘）来存储特别大的状态(state)。

这篇文档解释了如何在应用中使用Flink的状态(state)。

## 带有键值的状态（Keyed State）和算子状态（Operator State）

Flink中共有两类状态（state）:`带有键值的状态（Keyed State）` 和 `算子状态（Operator State）`。

### 带有键值的状态（Keyed State）

*带有键值的状态（Keyed State）*总是和键（keys）有关并且只能用在`带有键值的流（KeyedStream）`的函数和操作符中。

你可以认为带有键值的状态(Keyed State)像算子状态(Operator State)一样已经被
分区或者分片过了，每一个键值有唯一的一个状态（state）划分。每一个带有键值的状态（Keyed-state）逻辑上与
唯一的一对<并行的操作符实例，键值>(<parallel-operator-instance,key>)相连接，
并且由于每一个键又属于唯一的一个并行的带有键值的算子（keyed operator）的实例，我们可以简单的
认为带有键值的状态（Keyed-state）逻辑上与唯一的一对<操作符，键值>(<operator, key>)相连。

带有键值的状态（Keyed State）接着被放在*键组(Key Groups)*中进行管理。键组是Flink用来
重新分布带有键值的状态（Keyed State）的原子单元；并行化数量被设为了多少，就有多少个键值组。
在执行过程中每一个并行的带有键值的算子对应一个或多个键值组中的键值。

### 算子状态（Operator State）

用*算子状态*(*Operator State*)的时候，每一个算子状态（operator state）对
应一个并行的操作符实例。

Kafka 连接器[Kafka Connector](../connectors/kafka.html) 是在Flink中使用算子状态（operator state）的一个很好的
实例。每一个并行的Kafka消费组(Kafka consumer)里保存着那些主题分片(topic partition)的map和其偏移量(offsets)来作为算子状态(Operator state).

算子状态(Operator state）接口支持在并行化情况改变的时候对并行算子状态（state）进行重分布。我们还有不同的模式来支持重分布。

## 原生的与托管的状态（Raw and Managed State） 

*键值状态 (Keyed State)* 和 *算子状态(Operator State)* 存在两种形式： *托管的（managed)* and *原生的(raw)*.

*托管的状态(managed state)*表现为由Flink内部控制的一些数据结构，例如内部的哈希表，或者 RocksDB。对于"值状态（ValueState）"和"列表状态（Liststate）"这样的，Flink先内部对这些状态（State）进行编码，然后把它们写到检查点中。

*原生的状态 (Raw state)* 指的是算子可以保存着自己的数据结构的状态(state)。当被写成检查点的时候，它们只写入一些比特序列串，Flink对这些状态（State）的内部数据结构一无所知，只能看到那些原生的比特序列。

Flink提供的所有的数据流函数都可以运用托管的状态(managed state)，但是原生的状态(raw state)接口只能在实现算子的时候使用。比起原生的状态(raw state)，我们更加推荐使用托管的状态(managed state)，因为在并行化的情况改变的时候，Flink可以自动的重新分布托管的状态(managed state)，并且也能够更好的做到内存管理。


托管的键值状态（Managed Keyed State）接口提供了通过当前输入元素的键值来存取使用各种不同类型的状态（state)的方法。
这就意味着这种类型的状态（state)只能在`带有键值的流（KeyedStream）`中使用，我们可以通过`stream.keyBy(…)`来创建它。

现在，我们先来看看有哪些可供我们使用的不同类型的状态（state），然后再看如何在程序中使用它们。可以使用的状态（state)主要有以下几种：
 
* `ValueState<T>`: 这里面存储着一个可以被更新检索的状态(state)（与之前提过的输入元素的键值相关，所以对于每一个键值都
有可能有一个相应的状态(state)的值），这个值可以用`update（T）`来设置，用`T value()`来检索。

* `ListState<T>`: 这里存储着一个元素的列表。您可以向列表中增补元素，也可以用一个`Iterable`来检索当前所有的元素。
增补元素您可以使用`add(T)`, 迭代的检索元素您可以使用`Iterable<T> get()`。 

* `ReducingState<T>`: 这里面存储着一个单一的数值，它代表了所有被加到这个状态(state)里面的值共同聚合后的值。
它的接口和`ListState`是一样的，只是元素用`add(T)`加进状态（state)，用 `ReduceFunction`来做具体的聚合操作。

* `FoldingState<T, ACC>`: 这里存储着一个单一的数值，代表了所有被加到这个状态(state)里面的元素共同聚合后的值，与
`ReducingState` 不同的是，这个聚合的值的类型可以与被加进状态(state)中的值的类型不同。它的接口和`ListState`是
一样的，元素用`add(T)`加进状态（state)，但是用 `FoldFunction`来做具体的聚合操作。

* `MapState<UK, UV>`: 这里存储着一个mapping的列表，你可以将一个个键值对加入状态(state)然后用`Iterable`来检索当前存储的
所有的mappings,您可以使用`put(UK, UV)` 或者`putAll(Map<UK, UV>)`加入键值对，与键对应的数值可以用 `get(UK)`检索，
迭代的mappings,键和值可以分别用`entries()`, `keys()` 和 `values()`检索。

所有类型的状态(state)都可以用`clear()`来清除当前有效键（例如，当前输入元素的键）对应的状态(state)。 

<span class="label label-danger">注意！</span> 在未来的某一个Flink版本中我们将不再建议使用`FoldingState`，
它会被完全移除。我们将会提供一个更加有代表性的其他选择来方便大家使用。

您需要记住的第一点是您只能在状态(state)接口中使用这些状态对象，它们并非必须存储在内部也有可能在磁盘或者其他地方重置。
第二点是您得到的状态(state)的值是依赖于它所对应的键的，因此如果键不同的话那您在每一个函数调用中所得到的值也会不同。

为了对状态(state)进行操作，您必须要创建一个 `StateDescriptor`，它保存了这个状态(state)的名称，（接下来您会看到您
可以创建多个状态(state)，但是它们必须有相同的名字，以便你今后关联它们），还有状态(state)保存着的值的类型，有可能还有和一个
用户自定义的函数，比如说一个 `ReduceFunction`。您可以根据您想检索的状态(state)的类型，来决定您是创建一个`ValueStateDescriptor`,
还是一个`ListStateDescriptor`，或者是`ReducingStateDescriptor`或者`FoldingStateDescriptor` 或者 `MapStateDescriptor`。

您可以通过`RuntimeContext`来存取状态(state)，所以它只能用在*rich functions*中。请点击[这里]({{ site.baseurl }}/dev/api_concepts.html#rich-functions) 来获得更多信息，我们也先看一个小例子。 `RichFunction`中的
`RuntimeContext`有以下几个方法来存取状态(state)：

* `ValueState<T> getState(ValueStateDescriptor<T>)`
* `ReducingState<T> getReducingState(ReducingStateDescriptor<T>)`
* `ListState<T> getListState(ListStateDescriptor<T>)`
* `FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)`
* `MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)`

这里是一个`FlatMapFunction`的例子，让我们看看这几部分是如何协调工作的：

{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * 保存处理ValueState,第一项为计数值，第二项为不断累加的和。
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // 存取状态值
        Tuple2<Long, Long> currentSum = sum.value();

        // 更新计数值
        currentSum.f0 += 1;

        // 将输入的元组值累计到第二项中
        currentSum.f1 += input.f1;

        // 更新状态
        sum.update(currentSum);

        // 如果计数值达到2，计算平均数，清除状态值
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // 状态名
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // 状态类型信息
                        Tuple2.of(0L, 0L)); // 默认的状态值
        sum = getRuntimeContext().getState(descriptor);
    }
}

//在这里我们可以应用于数据流程序(假定我们有一个StreamExecutionEnvironment环境) 
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// 输出将为(1,4)和(1,5)
{% endhighlight %}

这个例子实现的是一个简单的窗口计数。我们把元组第一项作为键（在例子中所有的键都是1）。我们在方法中的
`ValueState`里面存储了一个计数值和一个不断累计着的总数和。一旦计数值达到2它就会计算平均数并且清除状态(state)
以便我们重新从`0`开始。需要注意的是如果我们的元组的第一项值不同的话，那么对于不同的键，状态(state)值也会不同。

### 在Scala数据流API使用state（State in the Scala DataStream API）

除了上述的接口以外，Scala API针对在有单一`ValueState`的`KeyedStream`上运用有状态的(stateful)的`map()` 
或者 `flatMap()`方法还有一个快捷方法。用户定义的函数在一个`Option`中得到`ValueState`的当前值，然后
返回一个用来更新状态(state)的已经更新的值。

{% highlight scala %}
val stream: DataStream[(String, Int)] = ...

val counts: DataStream[(String, Int)] = stream
  .keyBy(_._1)
  .mapWithState((in: (String, Int), count: Option[Int]) =>
    count match {
      case Some(c) => ( (in._1, c), Some(c + in._2) )
      case None => ( (in._1, 0), Some(in._2) )
    })
{% endhighlight %}

## 运用托管的算子状态（Using Managed Operator State）

要想运用托管的算子状态（managed operator state）,我们可以通过实现一个更一般化的`CheckpointedFunction`
接口或者实现`ListCheckpointed<T extends Serializable>`接口来达到我们的目的。

#### 检查点函数(CheckpointedFunction)

通过`CheckpointedFunction`接口我们可以存取一个无键的拥有重分布模式的状态(state)。它需要实现两个方法：

{% highlight java %}
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
{% endhighlight %}

无论何时要用到检查点，都要调用`snapshotState()`。而计数部分:`initializeState()`是在每次用户自定义函数初始化的时候被调用，无论是
函数第一次被初始化还是函数真的从之前的检查点恢复过来而进行的初始化，这个函数都要被调用。由此，`initializeState()`不仅仅是一个用来初始化不同
类型的状态(state)的地方，它里面还包含着状态(state)恢复逻辑。

现在，Flink已经支持列表形式的托管的算子状态(state)了。这个状态(state)是一个包含一些*seriablizable(序列化)*对象的列表，这些对象彼此相互独立,
因此很适合重新调整分布。换句话说，这些对象是非键值状态(non-keyed state)用来重分布的最小粒度。根据不同的存取状态(state)的方法，我们可以定义
一下的几种分布模式：

  - **Even-split redistribution（均分重分布）:** 每一个算子返回一个包含状态元素的列表。整个状态(state)是所有列表的一个串联.在恢复/重分布
    的时候，这个列表会均匀分布为多个子列表，数量与并行算子的数量相同。每个算子得到一个子列表，它可能为空，或者包含一个或多个元素。举个栗子，
    如果并行度为1的时候，这个算子的检查点状态(checkpointed state)含有元素1`element1`和元素2`element2`，那么当并行度增加至2时，元素1`element1`
    可能被分给算子0，而元素2`element2`可能会去算子1。

  - **Union redistribution（联合重分布）:** 每一个算子返回一个包含状态元素的列表。整个状态(state)是所有列表的一个串联。在恢复/重分布的时候，
    每个算子得到完整的包含状态元素的列表。

下面是一个有状态的(stateful)的`SinkFunction`运用`CheckpointedFunction`在将元素送出去之前先将它们缓存起来的例子。它表明了基本的均分重分布
(even-split redistribution)型列表状态(list state)：

{% highlight java %}
public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction,
                   CheckpointedRestoring<ArrayList<Tuple2<String, Integer>>> {

    private final int threshold;

    private transient ListState<Tuple2<String, Integer>> checkpointedState;

    private List<Tuple2<String, Integer>> bufferedElements;

    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<String, Integer> value) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // 传给data sink
            }
            bufferedElements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}),
                Tuple2.of(0L, 0L));
                
        checkpointedState = context.getOperatorStateStore().getListState(descriptor);

        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }

    @Override
    public void restoreState(ArrayList<Tuple2<String, Integer>> state) throws Exception {
        // 这个函数是来自于CheckpointedRestoring接口
        this.bufferedElements.addAll(state);
    }
}
{% endhighlight %}

`initializeState` 方法将`FunctionInitializationContext`作为参数，用来初始化非键值状态(non-keyed state)的"容器"。
`ListState`的这个容器用来根据检查点而存储非键值状态(non-keyed state)对象。

请注意一下状态(state)被初始化的方式，与键值状态(keyed state)类似，也是
用一个带有状态名字和状态存储的值的类型的`StateDescriptor`来初始化的。

{% highlight java %}
ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}),
        Tuple2.of(0L, 0L));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);
{% endhighlight %}

存取状态(state)的方法的命名规则是这样的：“重分布模式”后面加上“状态结构(state structure)"。例如，
如果在恢复时要用列表状态(list state)的联合重分布模式，就用`getUnionListState(descriptor)`来存取它，
如果名字里面没有包含重分布模式，*e.g.* `getListState(descriptor)`，这就意味着您将使用基本的均分重分布模式了。

初始化容器(container)过后，我们用环境中的`isRestored()`方法来检查我们是否在发生错误后做了恢复，例如如果结果为`true`，
那就说明我们做了恢复，恢复逻辑被应用了。

就像在代码中被修改过的`BufferingSink`例子展示的那样，这个在状态初始化时做了恢复的`ListState`被保存在一个类变量中
以便将来供`snapshotState()`使用。这样以来这个`ListState`中的所有元素，包括之前的检查点都被会被清除掉，然后用我们想
增加的新检查点来填充它。

补充说明一点，键值状态(keyed state)也可以用`initializeState()`方法来初始化， 这个可以用Flink提
供的`FunctionInitializationContext`来完成。

#### 列表检查点（ListCheckpointed）

`ListCheckpointed` 接口是`CheckpointedFunction`的一个具有更多约束的变量，它只支持在恢复时
使用列表类型的状态(state)并且是均分重分布模式的情况。它也需要实现以下两个方法：

{% highlight java %}
List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

void restoreState(List<T> state) throws Exception;
{% endhighlight %}

在`snapshotState()`方法中算子需要返回一个列表给检查点，在`restoreState`中要处理这个列表。如果状态(state)
没有被重新分片，那你总是可以在`snapshotState()`中返回`Collections.singletonList(MY_STATE)`。

### 带有状态的源函数（Stateful Source Functions）

带有状态的源(Stateful sources)和其他算子相比需要多注意一下。为了更新状态(state)并且保证输出原子化(需要在错误/恢复时
保证且只有一个语义(exactly-once semantics)),用户必须从源环境(source's context)中获得一个锁(lock)。

{% highlight java %}
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements ListCheckpointed<Long> {

    /**  目前距离实现有且只有一个语义（exactly once semantics）的偏差值 */
    private Long offset;

    /** 取消job的标识*/
    private volatile boolean isRunning = true;

    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();

        while (isRunning) {
            // output and state update are atomic
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

    @Override
    public List<Long> snapshotState(long checkpointId, long checkpointTimestamp) {
        return Collections.singletonList(offset);
    }

    @Override
    public void restoreState(List<Long> state) {
        for (Long s : state)
            offset = s;
    }
}
{% endhighlight %}

当Flink知道一个检查点在与外部进行通信时，一些算子可能需要一些信息。在这种情况下您可以查看`org.apache.flink.runtime.state.CheckpointListener`接口，来获得更多信息。
