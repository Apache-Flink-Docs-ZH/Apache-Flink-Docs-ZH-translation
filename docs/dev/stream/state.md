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

带有状态的（Stateful）函数与操作符可以在处理单个元素/事件的时候存储数据，使“状态（state）"成为任何类型的复杂操作的的关键组成部分。例如：
Stateful functions and operators store data across the processing of individual elements/events, making state a critical building block for
any type of more elaborate operation. For example:

  - 当我们需要建立一个搜索固定事件模式(events patterns)的应用时，状态（state）将可以存储至今为止所有事件的序列。When an application searches for certain event patterns, the state will store the sequence of events encountered so far.
  - 当需要每分钟聚合一次事件（events）的时候，state将会hold住那些未被处理的聚合。When aggregating events per minute, the state holds the pending aggregates.
  - 当为数据流训练一个机器学习模型的时候，state会存储当前时刻模型的参数。When training a machine learning model over a stream of data points, the state holds the current version of the model parameters.

为了使状态（state）具有容错性，Flink需要知道state的状态并对他做[检查点(checkpoint)](checkpointing.html).In order to make state fault tolerant, Flink needs to be aware of the state and [checkpoint](checkpointing.html) it.
在许多情况下，Flink还支持在应用中*管理*state,这就意味着Flink需要用到内存管理（或者内存不够时写入磁盘）来存储特别大的state。In many cases, Flink can also *manage* the state for the application, meaning Flink deals with the memory management (possibly spilling to disk
if necessary) to allow applications to hold very large state.

这篇文档解释了如何在应用中使用Flink的state抽象。This document explains how to use Flink's state abstractions when developing an application.


## 带有键值的状态（Keyed State） and 算子状态（Operator State）

Flink中共有两类状态（state）:`带有键值的状态（Keyed State）` 和 `算子状态（Operator State）`。 There are two basic kinds of state in Flink: `Keyed State` and `Operator State`.

### 带有键值的状态（Keyed State）

*带有键值的状态（Keyed State）*总是和键（keys）有关并且只能用在`带有键值的流（KeyedStream）`的函数和操作符中。*Keyed State* is always relative to keys and can only be used in functions and operators on a `KeyedStream`.

你可以认为带有键值的状态(Keyed State)像算子状态(Operator State)一样已经被
分区或者分片过了，每一个键值有唯一的一个状态（state）划分。每一个带有键值的状态（Keyed-state）逻辑上与
唯一的一对<并行的操作符实例，键值>(<parallel-operator-instance,key>)相连接，
并且由于每一个键又属于唯一的一个并行的带有键值的算子（keyed operator）的实例，我们可以简单的
认为带有键值的状态（Keyed-state）逻辑上与唯一的一对<操作符，键值>相连。
You can think of Keyed State as Operator State that has been partitioned,
or sharded, with exactly one state-partition per key.
Each keyed-state is logically bound to a unique
composite of <parallel-operator-instance, key>, and since each key
"belongs" to exactly one parallel instance of a keyed operator, we can
think of this simply as <operator, key>.

带有键值的状态（Keyed State）接着被放在*键组(Key Groups)*中进行管理。键组是Flink用来
重新分布带有键值的状态（Keyed State）的原子单元；并行化被设为了多少，就实际上有多少个键值组。
在执行过程中每一个并行的带有键值的操作符实例对应一个或多个键值组中的键值。
.Keyed State is further organized into so-called *Key Groups*. Key Groups are the
atomic unit by which Flink can redistribute Keyed State;
there are exactly as many Key Groups as the defined maximum parallelism.
During execution each parallel instance of a keyed operator works with the keys
for one or more Key Groups.

### 算子状态（Operator State）

用*算子状态*(或者叫做*Operator State*)的时候，每一个算子状态（operator state）对
应一个并行的操作符实例。
With *Operator State* (or *non-keyed state*), each operator state is
bound to one parallel operator instance.

Kafka 连接器[Kafka Connector](../connectors/kafka.html) 是在Flink中使用算子状态（operator state）的一个很好的
实例。每一个并行的Kafka消费组(Kafka consumer)里存储着那些主题分片的map和偏移量来作为操作符state.
 is a good motivating example for the use of Operator State
in Flink. Each parallel instance of the Kafka consumer maintains a map
of topic partitions and offsets as its Operator State.

算子状态(Operator state）接口支持在并行化情况改变的时候对并行算子实例重分布状态（state）。做这样的重分布有很多中方案。
The Operator State interfaces support redistributing state among
parallel operator instances when the parallelism is changed. There can be different schemes for doing this redistribution.

## 原生的与管理的状态（Raw and Managed State） 

*键值状态 (Keyed State)* 和 *算子状态(Operator State)* 存在两种形式： *管理的（managed)* and *原生的(raw)*.
*Keyed State* and *Operator State* exist in two forms: *managed* and *raw*.

在数据结构中存在的*管理的状态*(managed state)是由Flink内部运行来控制的，例如内部的哈希表，或者 RocksDB。像"值状态（ValueState）"和"列表状态（Liststate）"等等都属于这种类型。 Flink 内部运行对这些状态（State）进行编码，然后把它们写到检查点中。
 is represented in data structures controlled by the Flink runtime, such as internal hash tables, or RocksDB.
Examples are "ValueState", "ListState", etc. Flink's runtime encodes
the states and writes them into the checkpoints.

*原生的状态 (Raw state)* 指的是那些保存着它们自己的数据结构的操作符(算子)。当被写成检查点的时候，它们只写入一些比特序列串，Flink对这些状态（State）的内部数据结构一无所知，只能看到那些原生的比特序列。
is state that operators keep in their own data structures. When checkpointed, they only write a sequence of bytes into
the checkpoint. Flink knows nothing about the state's data structures and sees only the raw bytes.

Flink提供的所有的数据流函数都可以运用管理的状态(managed state)，但是原生的状态(raw state)接口只能在实现算子的时候使用。比起原生的状态(raw state)，我们更加推荐使用管理的状态(managed state)，因为在并行化的情况改变的时候，Flink可以自动的重新分布管理的状态(managed state)，并且也能够更好的做到内存管理。
All datastream functions can use managed state, but the raw state interfaces can only be used when implementing operators.
Using managed state (rather than raw state) is recommended, since with
managed state Flink is able to automatically redistribute state when the parallelism is
changed, and also do better memory management.

## 运用管理的键值状态（Using Managed Keyed State）

管理的键值状态（Managed Keyed State）接口提供了通过当前输入元素的键值来存取使用各种不同类型的状态（state)的方法。
这就意味着这种类型的状态（state)只能用在`带有键值的流（KeyedStream）`中，我们可以通过`stream.keyBy(…)`来创建它。
The managed keyed state interface provides access to different types of state that are all scoped to
the key of the current input element. This means that this type of state can only be used
on a `KeyedStream`, which can be created via `stream.keyBy(…)`.

现在，我们先来看看有哪些可供我们使用的不同类型的状态（state），然后再看如何在程序中使用它们。可以使用的状态（state)主要有以下几种：
Now, we will first look at the different types of state available and then we will see
how they can be used in a program. The available state primitives are:
 
* `ValueState<T>`: 这里面存储着一个可以被更新检索的状态(state)（与之前提过的输入元素的键值相关，所以根据实际的情况，对于每一个键值都有可能有一个相应的值），这个值可以用`update（T）`来设置，用`T value()`来检索。
This keeps a value that can be updated and
retrieved (scoped to key of the input element as mentioned above, so there will possibly be one value
for each key that the operation sees). The value can be set using `update(T)` and retrieved using
`T value()`.

* `ListState<T>`: 这里存储着一个元素的列表。您可以向列表中增补元素，也可以用一个`Iterable`来检索当前所有的元素。
增补元素您可以使用`add(T)`, 迭代的检索元素您可以使用`Iterable<T> get()`。 
This keeps a list of elements. You can append elements and retrieve an `Iterable`
over all currently stored elements. Elements are added using `add(T)`, the Iterable can
be retrieved using `Iterable<T> get()`.

* `ReducingState<T>`: 这里面存储着一个单一的数值，它代表了所有被加到这个状态(state)里面的值共同聚合后的值。
它的接口和`ListState`是一样的，只是元素用`add(T)`加进状态（state)，用 `ReduceFunction`来做具体的聚合操作。
This keeps a single value that represents the aggregation of all values
added to the state. The interface is the same as for `ListState` but elements added using
`add(T)` are reduced to an aggregate using a specified `ReduceFunction`.

* `FoldingState<T, ACC>`: 这里存储着一个单一的数值，代表了所有被加到这个状态(state)里面的元素共同聚合后的值，与
`ReducingState` 不同的是，这个聚合的值的类型也许与被加进状态(state)中的值的类型不同。它的接口和`ListState`是
一样的，只是元素用`add(T)`加进状态（state)，用 `FoldFunction`来做具体的聚合操作。
This keeps a single value that represents the aggregation of all values added to the state. Contrary to `ReducingState`, the aggregate type may be different from the type
of elements that are added to the state. The interface is the same as for `ListState` but elements
added using `add(T)` are folded into an aggregate using a specified `FoldFunction`.

* `MapState<UK, UV>`: 这里存储着一个mapping的列表，你可以将一个个键值对加入状态(state)然后用`Iterable`来检索当前存储的所有的mappings,您可以使用`put(UK, UV)` 或者`putAll(Map<UK, UV>)`加入键值对，与键对应的数值可以用 `get(UK)`检索，迭代的mappings,键和值可以分别用`entries()`, `keys()` 和 `values()`检索。
This keeps a list of mappings. You can put key-value pairs into the state and
retrieve an `Iterable` over all currently stored mappings. Mappings are added using `put(UK, UV)` or 
`putAll(Map<UK, UV>)`. The value associated with a user key can be retrieved using `get(UK)`. The iterable
views for mappings, keys and values can be retrieved using `entries()`, `keys()` and `values()` respectively.

所有类型的状态(state)都可以用`clear()`来清除当前有效键（例如，当前输入元素的键）对应的状态(state)。 
All types of state also have a method `clear()` that clears the state for the currently
active key, i.e. the key of the input element.

<span class="label label-danger">Attention</span> 在未来的某一个Flink版本中我们将不再建议使用`FoldingState`，
它会被完全移除。我们将会提供一个更加有代表性的选择来方便大家使用。
will be deprecated in one of
the next versions of Flink and will be completely removed in the future. A more general
alternative will be provided.

您需要记住的第一点是您只能在状态(state)接口中使用这些状态对象，它们并非必须存储在内部却有可能在磁盘或者其他地方重置。
第二点是您得到的状态(state)的值是依赖于它所对应的键的，因此如果键不同的话那您在每一个函数调用中所得到的值也会不同。
It is important to keep in mind that these state objects are only used for interfacing
with state. The state is not necessarily stored inside but might reside on disk or somewhere else.
The second thing to keep in mind is that the value you get from the state
depends on the key of the input element. So the value you get in one invocation of your
user function can differ from the value in another invocation if the keys involved are different.

为了对状态(state)进行操作，您必须要创建一个 `StateDescriptor`，它保存了这个状态(state)的名称，（接下来您会看到您
可以创建多个状态(state)，但是它们必须有相同的名字，以便你今后关联它们），状态(state)保存着的值的类型，有可能还有和一个
用户自定义的函数，比如说一个 `ReduceFunction`。您可以根据您想检索的状态(state)的类型，来决定您是创建一个`ValueStateDescriptor`,
还是一个`ListStateDescriptor`，或者是`ReducingStateDescriptor`或者`FoldingStateDescriptor` 或者 `MapStateDescriptor`。
To get a state handle, you have to create a `StateDescriptor`. This holds the name of the state
(as we will see later, you can create several states, and they have to have unique names so
that you can reference them), the type of the values that the state holds, and possibly
a user-specified function, such as a `ReduceFunction`. Depending on what type of state you
want to retrieve, you create either a `ValueStateDescriptor`, a `ListStateDescriptor`,
a `ReducingStateDescriptor`, a `FoldingStateDescriptor` or a `MapStateDescriptor`.

您可以通过`RuntimeContext`来存取状态(state)，所以它只能用在*rich functions*中。请点击[这里]({{ site.baseurl }}/dev/api_concepts.html#rich-functions) 来获得更多信息，但我们也先简短地看一个小例子， `RichFunction`中的
`RuntimeContext`有以下几个方法来存取状态(state)：
State is accessed using the `RuntimeContext`, so it is only possible in *rich functions*.
Please see [here]({{ site.baseurl }}/dev/api_concepts.html#rich-functions) for
information about that, but we will also see an example shortly. The `RuntimeContext` that
is available in a `RichFunction` has these methods for accessing state:

* `ValueState<T> getState(ValueStateDescriptor<T>)`
* `ReducingState<T> getReducingState(ReducingStateDescriptor<T>)`
* `ListState<T> getListState(ListStateDescriptor<T>)`
* `FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)`
* `MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)`

这里是一个`FlatMapFunction`的例子，让我们看看这几部分是如何协调工作的：
that shows how all of the parts fit together:

{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();

        // update the count
        currentSum.f0 += 1;

        // add the second field of the input value
        currentSum.f1 += input.f1;

        // update the state
        sum.update(currentSum);

        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                        Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}

// this can be used in a streaming program like this (assuming we have a StreamExecutionEnvironment env)
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
{% endhighlight %}

这个例子实现的是一个简单的窗口计数。我们把元组第一列作为键（在例子中所有的键都是1）。方法中存储着计数，
在`ValueState`里面不断累计着总数。一旦计数值达到2它就会计算平均数并且清除状态(state)以便我们重新从`0`开始。
需要注意的是如果我们的元组的第一个元素拥有不同的值的话，那么对于不同的键，也会保存着不同的状态值。
This example implements a poor man's counting window. We key the tuples by the first field
(in the example all have the same key `1`). The function stores the count and a running sum in
a `ValueState`. Once the count reaches 2 it will emit the average and clear the state so that
we start over from `0`. Note that this would keep a different state value for each different input
key if we had tuples with different values in the first field.

### State in the Scala DataStream API

除了上述的接口以外，Scala API对于`KeyedStream`上带有单一`ValueState`的有状态的(stateful)的`map()` 
或者 `flatMap()`方法有一个快捷方法。用户定义的函数在一个`Option`中得到当前`ValueState`的当前值，然后
返回一个用来更新状态(state)的值。
In addition to the interface described above, the Scala API has shortcuts for stateful
`map()` or `flatMap()` functions with a single `ValueState` on `KeyedStream`. The user function
gets the current value of the `ValueState` in an `Option` and must return an updated value that
will be used to update the state.

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

## 运用管理的算子状态(state) Using Managed Operator State

要想运用管理的算子状态（managed operator state）,我们可以通过实现一个更一般化的`CheckpointedFunction`
接口或者实现`ListCheckpointed<T extends Serializable>`接口来达到我们的目的。
To use managed operator state, a stateful function can implement either the more general `CheckpointedFunction`
interface, or the `ListCheckpointed<T extends Serializable>` interface.

#### 检查点函数(CheckpointedFunction)

通过`CheckpointedFunction`接口我们可以存取一个无键的拥有重分布模式的状态(state)。它需要实现两个方法：
The `CheckpointedFunction` interface provides access to non-keyed state with different
redistribution schemes. It requires the implementation of two methods:

{% highlight java %}
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
{% endhighlight %}

无论何时要用到检查点，都要调用`snapshotState()`。而计数部分:`initializeState()`是在每次用户自定义函数初始化的时候被调用，无论是
函数第一次被初始化还是函数真的从之前的检查点恢复过来而初始化，都要被调用。由此，`initializeState()`不仅仅是一个用来初始化不同
类型的状态(state)的地方，它里面还包含着状态(state)恢复逻辑。
Whenever a checkpoint has to be performed, `snapshotState()` is called. The counterpart, `initializeState()`,
is called every time the user-defined function is initialized, be that when the function is first initialized
or be that when the function is actually recovering from an earlier checkpoint. Given this, `initializeState()` is not
only the place where different types of state are initialized, but also where state recovery logic is included.

现在，Flink已经支持列表形式的管理的算子状态(state)了。这个状态(state)是一个一些*seriablizable(序列化)*对象的列表，这些对象彼此相互独立,
因此很适合重新调整分布。换句话说，这些对象时非键值状态(non-keyed state)用来重分布的最小粒度。根据不同的存取状态(state)的方法，我们可以定义
一下的几种分布模式：
Currently, list-style managed operator state is supported. The state
is expected to be a `List` of *serializable* objects, independent from each other,
thus eligible for redistribution upon rescaling. In other words, these objects are the finest granularity at which
non-keyed state can be redistributed. Depending on the state accessing method,
the following redistribution schemes are defined:

  - **Even-split redistribution（均分重分布）:** 每一个算子返回一个包含状态元素的列表。整个状态(state)是所有列表的一个串联.在恢复/重分布
    的时候，这个列表会均匀分布为多个子列表，数量与并行算子的数量相同。每个算子得到一个子列表，它可能为空，或者包含一个或多个元素。举个栗子，
    如果并行度为1的时候，这个算子的检查点状态(checkpointed state)含有元素1`element1`和元素2`element2`，那么当并行度增加至2时，元素1`element1`
    可能被分给算子0，而元素2`element2`可能会取算子1。
    Each operator returns a List of state elements. The whole state is logically a concatenation of
    all lists. On restore/redistribution, the list is evenly divided into as many sublists as there are parallel operators.
    Each operator gets a sublist, which can be empty, or contain one or more elements.
    As an example, if with parallelism 1 the checkpointed state of an operator
    contains elements `element1` and `element2`, when increasing the parallelism to 2, `element1` may end up in operator instance 0,
    while `element2` will go to operator instance 1.

  - **Union redistribution（联合重分布）:** 每一个算子返回一个包含状态元素的列表。整个状态(state)是所有列表的一个串联。在恢复/重分布的时候，
    每个算子得到完整的包含状态元素的列表。
    Each operator returns a List of state elements. The whole state is logically a concatenation of
    all lists. On restore/redistribution, each operator gets the complete list of state elements.

下面是一个有状态的(stateful)的`SinkFunction`运用`CheckpointedFunction`在将元素送出去之前先将它们缓存起来的例子。它表明了基本的均分重分布
(even-split redistribution)型列表状态(list state)：
Below is an example of a stateful `SinkFunction` that uses `CheckpointedFunction`
to buffer elements before sending them to the outside world. It demonstrates
the basic even-split redistribution list state:

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
                // send it to the sink
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
        // this is from the CheckpointedRestoring interface.
        this.bufferedElements.addAll(state);
    }
}
{% endhighlight %}

`initializeState` 方法将`FunctionInitializationContext`作为参数，用来初始化非键值状态(non-keyed state)的"容器"。
这个容器属于`ListState`, 用来依据检查点存储非键值状态(non-keyed state)对象。
The `initializeState` method takes as argument a `FunctionInitializationContext`. This is used to initialize
the non-keyed state "containers". These are a container of type `ListState` where the non-keyed state objects
are going to be stored upon checkpointing.

请注意一下状态(state)被初始化的方式，与键值状态(keyed state)类似，也是
用一个带有状态名字和状态存储的值的类型的`StateDescriptor`来初始化的。
Note how the state is initialized, similar to keyed state,
with a `StateDescriptor` that contains the state name and information
about the type of the value that the state holds:

{% highlight java %}
ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}),
        Tuple2.of(0L, 0L));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);
{% endhighlight %}

The naming convention of the state access methods contain its redistribution
pattern followed by its state structure. For example, to use list state with the
union redistribution scheme on restore, access the state by using `getUnionListState(descriptor)`.
If the method name does not contain the redistribution pattern, *e.g.* `getListState(descriptor)`,
it simply implies that the basic even-split redistribution scheme will be used.

After initializing the container, we use the `isRestored()` method of the context to check if we are
recovering after a failure. If this is `true`, *i.e.* we are recovering, the restore logic is applied.

As shown in the code of the modified `BufferingSink`, this `ListState` recovered during state
initialization is kept in a class variable for future use in `snapshotState()`. There the `ListState` is cleared
of all objects included by the previous checkpoint, and is then filled with the new ones we want to checkpoint.

As a side note, the keyed state can also be initialized in the `initializeState()` method. This can be done
using the provided `FunctionInitializationContext`.

#### ListCheckpointed

The `ListCheckpointed` interface is a more limited variant of `CheckpointedFunction`,
which only supports list-style state with even-split redistribution scheme on restore.
It also requires the implementation of two methods:

{% highlight java %}
List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

void restoreState(List<T> state) throws Exception;
{% endhighlight %}

On `snapshotState()` the operator should return a list of objects to checkpoint and
`restoreState` has to handle such a list upon recovery. If the state is not re-partitionable, you can always
return a `Collections.singletonList(MY_STATE)` in the `snapshotState()`.

### Stateful Source Functions

Stateful sources require a bit more care as opposed to other operators.
In order to make the updates to the state and output collection atomic (required for exactly-once semantics
on failure/recovery), the user is required to get a lock from the source's context.

{% highlight java %}
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements ListCheckpointed<Long> {

    /**  current offset for exactly once semantics */
    private Long offset;

    /** flag for job cancellation */
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

Some operators might need the information when a checkpoint is fully acknowledged by Flink to communicate that with the outside world. In this case see the `org.apache.flink.runtime.state.CheckpointListener` interface.

