---
title: "Queryable State"
nav-parent_id: streaming
nav-pos: 61
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

<div class="alert alert-warning">
  <strong>注：</strong> 可查询状态的API当前处于发展状态，所以<strong>不保证</strong> 接口的
  稳定性。即将到来的版本中API有可能会发生重大变化。
</div>

简而言之，此功能允许用户从外部查询flink托管的分区的状态(see [使用状态]({{ site.baseurl }}/dev/stream/state.html))。在某些场景，可查询状态取消了需要和外部系统进行分布式交互的依赖，例如在实践中经常是瓶颈的键值存储。

<div class="alert alert-warning">
  <strong>注意:</strong> 可查询状态并发访问键值状态（keyed state）比同步访问更有可能阻碍其操作。因为某些状态存储使用的是java的堆空间，例如：内存状态存储，
   Fs状态存储，所以内存不直接使用复制当检索值,而是引用存储值,可能会导致读-修改-写模式是不安全的，
   且并发修改将会导致可查询状态服务失败。对于这些问题RocksDB状态存储是安全的。
</div>

## 使用可查询状态
  为了使用可查询状态，通过设置配置参数 `query.server.enable` 为 `true`(当前的默认值)，是全局
激活可查询状态服务的第一步。然后某些的状态需要可查询可以通过
* `QueryableStateStream`, 可对流入的值进行状态查询，或
* `StateDescriptor#setQueryable(String queryableStateName)`，使某个算子的键值状态可查询。

下面的段落解释这两种方法的使用。

### 可查询状态流

`键值流` 可以通过下面的方法可以将他的值作为可查询状态:

{% highlight java %}
// ValueState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ValueStateDescriptor stateDescriptor)

// Shortcut for explicit ValueStateDescriptor variant
QueryableStateStream asQueryableState(String queryableStateName)

// FoldingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    FoldingStateDescriptor stateDescriptor)

// ReducingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ReducingStateDescriptor stateDescriptor)
{% endhighlight %}


<div class="alert alert-info">
  <strong>注:</strong> 不存在 <code>ListState</code> sink 由于不断增长的列表可能不可以清理，
  将会占用大量内存。
</div>

调用这些方法将会返回`QueryableStateStream`，无法再做其他操作， 且当前只保存可查询状态的名称以及
值和键序列化的可查询状态流。它与水槽（sinlk）相当，不能再进一步转变。

在内部一个 `QueryableStateStream` 被翻译成一个使用所有流入记录来更新可查询状态的算子的实例。

下面的项目，所有键值流的记录将会被用来更新状态实例，通过`ValueState#update(value)`或者
`AppendingState#add(value)`， 依赖于所选状态的转化：
{% highlight java %}
stream.keyBy(0).asQueryableState("query-name")
{% endhighlight %}
这样做就像Scala API的“flatMapWithState”。

### 托管的键值状态

一个算子的被管理键值状态
(see [使用托管的键值状态]({{ site.baseurl }}/dev/stream/state.html#using-managed-keyed-state))
可以通过使适当的状态描述符可查询来进行查询，例如下面的例子通过`StateDescriptor#setQueryable(String queryableStateName)`：
{% highlight java %}
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
                "average", // the state name
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
descriptor.setQueryable("query-name"); // queryable state name
{% endhighlight %}
<div class="alert alert-info">
  <strong>注:</strong> `queryableStateName`参数可以任意选择，且只能
   用于查询。它不一定与状态自己的名字相同。
</div>


## 查询状态

`QueryableStateClient`帮助器类可用于针对内部服务状态的“KvState”实例的查询。它需要设置一个有效的 `JobManager`
地址和端口, 创建方式如下：

{% highlight java %}
final Configuration config = new Configuration();
config.setString(ConfigConstants.JOB_MANAGER_IPC_ADDRESS_KEY, queryAddress);
config.setInteger(ConfigConstants.JOB_MANAGER_IPC_PORT_KEY, queryPort);

QueryableStateClient client = new QueryableStateClient(config);
{% endhighlight %}

查询的方法是:

{% highlight java %}
Future<byte[]> getKvState(
    JobID jobID,
    String queryableStateName,
    int keyHashCode,
    byte[] serializedKeyAndNamespace)
{% endhighlight %}

对这个方法的调用返回一个`Future`，最终持有ID为“jobID”的job，它的“queryableStateName”标识的可查询状态实例拥
有的序列化状态值。`keyHashCode`是由`Object.hashCode（）'返回的键的哈希码，而`serializedKeyAndNamespace`
是序列化的键和命名空间。
<div class="alert alert-info">
  <strong>注:</strong> 客户端是异步的，可以由多个线程共享。可通过<code>QueryableStateClient#shutdown()</code>
  关闭来释放资源，当不需要的时候。
</div>

目前的实现仍然是比较低级别的，因为它只适用于此用于提供键/命名空间和返回结果的序列化数据。用户（或一些后续实用程序）
需要为此设置序列化程序。这样做的好处是查询服务不需要担心任何类加载问题等。

有一些序列化帮助方法用于键/命名空间和值序列化在`KvStateRequestSerializer`里。

### 案例

下面的案例继承了 `CountWindowAverage`， 案例
(see [使用托管的键值状态]({{ site.baseurl }}/dev/stream/state.html#using-managed-keyed-state))
如何使它可查询，且如何查询这个值:

{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    private transient ValueState<Tuple2<Long /* count */, Long /* sum */>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {
        Tuple2<Long, Long> currentSum = sum.value();
        currentSum.f0 += 1;
        currentSum.f1 += input.f1;
        sum.update(currentSum);

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
        descriptor.setQueryable("query-name");
        sum = getRuntimeContext().getState(descriptor);
    }
}
{% endhighlight %}

一旦在job中使用，您可以检索作业ID，然后从该算子中查询任何键的当前状态：

{% highlight java %}
final Configuration config = new Configuration();
config.setString(ConfigConstants.JOB_MANAGER_IPC_ADDRESS_KEY, queryAddress);
config.setInteger(ConfigConstants.JOB_MANAGER_IPC_PORT_KEY, queryPort);

QueryableStateClient client = new QueryableStateClient(config);

final TypeSerializer<Long> keySerializer =
        TypeInformation.of(new TypeHint<Long>() {}).createSerializer(new ExecutionConfig());
final TypeSerializer<Tuple2<Long, Long>> valueSerializer =
        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}).createSerializer(new ExecutionConfig());

final byte[] serializedKey =
        KvStateRequestSerializer.serializeKeyAndNamespace(
                key, keySerializer,
                VoidNamespace.INSTANCE, VoidNamespaceSerializer.INSTANCE);

Future<byte[]> serializedResult =
        client.getKvState(jobId, "query-name", key.hashCode(), serializedKey);

// now wait for the result and return it
final FiniteDuration duration = new FiniteDuration(1, TimeUnit.SECONDS);
byte[] serializedValue = Await.result(serializedResult, duration);
Tuple2<Long, Long> value =
        KvStateRequestSerializer.deserializeValue(serializedValue, valueSerializer);
{% endhighlight %}

### Scala用户注意事项

创建“TypeSerializer”实例时，请使用可用的Scala扩展。 添加以下导入：

```scala
import org.apache.flink.streaming.api.scala._
```

现在你可以通过下面的方式创建 type serializers：

```scala
val keySerializer = createTypeInformation[Long]
  .createSerializer(new ExecutionConfig)
```

如果你不这样做，则可能会遇到Flink作业和客户端代码中使用的序列化器之间的不匹配，因为类似于`scala.Long`
的类型在运行时无法获取。

## 配置

以下配置参数会影响可查询状态服务器和客户端的行为。它们定义在“QueryableStateOptions”中。

### 服务端
* `query.server.enable`: 标记是否开启可查询状态服务端
* `query.server.port`: 端口绑定到内部的`KvStateServer`（0 => 选择随机可用端口）
* `query.server.network-threads`: `KvStateServer`（0 => #slots）的网络（事件循环）线程数
* `query.server.query-threads`: `KvStateServerHandler`（0 => #slots）的异步查询线程数。

### 客户端 (`QueryableStateClient`)
* `query.client.network-threads`: `KvStateClient`（0 =>可用内核数）的网络（事件循环）线程数
* `query.client.lookup.num-retries`: 位置查找失败次数。
* `query.client.lookup.retry-delay`: 位置查找失败重试延时间隔（毫秒）

## 限制

* 可查询状态的生命周期受限于job的生命周期，例如， 任务启动时注册可查询状态，并在清理时注销它。在将来的
版本中，最好是将其解耦，以便在任务完成后允许查询，并通过状态加速恢复通过状态复制。
* 有关KvState的通知可以通过一个简单的说明来实现。 未来这个应该需要改善，以便实现更强大的询问和确认。
* 服务器和客户端跟踪查询的统计信息。 因为它们不会在任何地方暴露出来，所以默认是禁用的。 一旦可以通过度
量系统更好的发布这些数字，我们应该启用统计。
