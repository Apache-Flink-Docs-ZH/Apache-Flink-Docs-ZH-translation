
---
title: "状态后端"
nav-parent_id: setup
nav-pos: 11
---
---
title: "State Backends"
nav-parent_id: setup
nav-pos: 11
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

用 [Data Stream API]({{ site.baseurl }}/dev/datastream_api.html) 编写的流处理程序，通常要保存不同形式的状态信息：
Programs written in the [Data Stream API]({{ site.baseurl }}/dev/datastream_api.html) often hold state in various forms:

- 窗口被触发之前收集或聚合的元素
- 转换函数也许用到key/value状态接口来存储values
- 转换函数也许实现了Checkpointed接口，使得本地变量具有容错的能力

- Windows gather elements or aggregates until they are triggered
- Transformation functions may use the key/value state interface to store values
- Transformation functions may implement the `Checkpointed` interface to make their local variables fault tolerant

可以在流处理API中查看 [Working with State]({{ site.baseurl }}/dev/stream/state.html).
See also [Working with State]({{ site.baseurl }}/dev/stream/state.html) in the streaming API guide.

当检查点功能被激活后，以上的这些基于特定检查点的状态将被持久化，以防止数据丢失或恢复到一致性状态。状态在内部如何被描述，如何被持久化以及持久化到哪里，这依赖于选择的**状态后端**。
When checkpointing is activated, such state is persisted upon checkpoints to guard against data loss and recover consistently.
How the state is represented internally, and how and where it is persisted upon checkpoints depends on the
chosen **State Backend**.

* ToC
{:toc}

## 可用的状态后端
## Available State Backends

开箱即用，Flink提供了这些状态后端：

 - *MemoryStateBackend*
 - *FsStateBackend*
 - *RocksDBStateBackend*
 
不作任何配置的情况下，系统默认使用MemoryStateBackend。

Out of the box, Flink bundles these state backends:

 - *MemoryStateBackend*
 - *FsStateBackend*
 - *RocksDBStateBackend*

If nothing else is configured, the system will use the MemoryStateBackend.


### MemoryStateBackend
### The MemoryStateBackend

*MemoryStateBackend*以对象的形式保存数据到java堆中。Key/value状态和窗口运算中的数据，用hash表来存储值、触发器等信息。
The *MemoryStateBackend* holds data internally as objects on the Java heap. Key/value state and window operators hold hash tables
that store the values, triggers, etc.

基于checkpoints接口的方式，状态后端将对状态进行快照，并作为检查点的一部分，发送通知给JobManager (master)，这些数据同样存储在java堆中。
Upon checkpoints, this state backend will snapshot the state and send it as part of the checkpoint acknowledgement messages to the
JobManager (master), which stores it on its heap as well.

MemoryStateBackend的限制：
Limitations of the MemoryStateBackend:

  - 每个独立的状态大小上限是5MB，这个值可以通过MemoryStateBackend的构造函数来增加。
  - 不考虑可配置的状态的最大值，状态的大小不能大于akka框架的。 (看 [Configuration]({{ site.baseurl }}/setup/config.html)).
  - 聚合的状态必须匹配到JobManager的内存.

  - The size of each individual state is by default limited to 5 MB. This value can be increased in the constructor of the MemoryStateBackend.
  - Irrespective of the configured maximal state size, the state cannot be larger than the akka frame size (see [Configuration]({{ site.baseurl }}/setup/config.html)).
  - The aggregate state must fit into the JobManager memory.

MemoryStateBackend鼓励用于:
The MemoryStateBackend is encouraged for:

  - 本地开发或调试
  - 拥有很小状态的Jobs, 例如那种一次包含一个记录的函数 (Map, FlatMap, Filter, ...). Kafka Consumer 就需要很小的状态.

  - Local development and debugging
  - Jobs that do hold little state, such as jobs that consist only of record-at-a-time functions (Map, FlatMap, Filter, ...). The Kafka Consumer requires very little state.


### FsStateBackend
### The FsStateBackend

*FsStateBackend* 可以通过一个文件系统的URL来配置 (type, address, path), 例如 "hdfs://namenode:40010/flink/checkpoints" 或者 "file:///data/flink/checkpoints".
The *FsStateBackend* is configured with a file system URL (type, address, path), such as "hdfs://namenode:40010/flink/checkpoints" or "file:///data/flink/checkpoints".


FsStateBackend保存TaskManager内存中的运行数据. 基于检查点功能, 它将包含状态的快照信息写入配置好的文件系统目录中. 最小的元数据信息被存储在JobManager的内存中 (或者在高可用模式下, 存储到元数据检查点中).
The FsStateBackend holds in-flight data in the TaskManager's memory. Upon checkpointing, it writes state snapshots into files in the configured file system and directory. Minimal metadata is stored in the JobManager's memory (or, in high-availability mode, in the metadata checkpoint).

FsStateBackend鼓励用于:
The FsStateBackend is encouraged for:

  - 拥有大的状态的Jobs, 很长的窗口, 很大的key/value状态.
  - 所有高可用安装模式.
  - Jobs with large state, long windows, large key/value states.
  - All high-availability setups.

### RocksDBStateBackend
### The RocksDBStateBackend

*RocksDBStateBackend*可以通过一个文件系统的URL来配置 (type, address, path), 例如 "hdfs://namenode:40010/flink/checkpoints" 或者 "file:///data/flink/checkpoints".
The *RocksDBStateBackend* is configured with a file system URL (type, address, path), such as "hdfs://namenode:40010/flink/checkpoints" or "file:///data/flink/checkpoints".

RocksDBStateBackend保存TaskManager中的运行数据，存储到[RocksDB](http://rocksdb.org) 数据库中. 基于检查点功能, 所有RocksDB数据库的数据将通过检查点保存到文件系统目录中. 最小的元数据信息被存储到JobManager的内存中 (或者在高可用模式下, 存储到元数据检查点中).
The RocksDBStateBackend holds in-flight data in a [RocksDB](http://rocksdb.org) data base
that is (per default) stored in the TaskManager data directories. Upon checkpointing, the whole
RocksDB data base will be checkpointed into the configured file system and directory. Minimal
metadata is stored in the JobManager's memory (or, in high-availability mode, in the metadata checkpoint).

RocksDBStateBackend鼓励用于:
The RocksDBStateBackend is encouraged for:

  - 拥有非常大状态的Jobs, 很长的窗口, 很大的key/value状态.
  - 所有高可用安装模式.

注意，RocksDBStateBackend状态的大小仅仅受限于磁盘空间的可用度。
相对于存储状态到内存的FsStateBackend，这允许保存非常大的状态。
这同样意味着，可以达到的最大的吞吐量将低于状态后端的。
Note that the amount of state that you can keep is only limited by the amount of disc space available.
This allows keeping very large state, compared to the FsStateBackend that keeps state in memory.
This also means, however, that the maximum throughput that can be achieved will be lower with
this state backend.

## 配置一个状态后端
## Configuring a State Backend

可以为每个job配置状态后端。而且，你可以定义一个默认的状态后端，以供当在job中没有明确定义状态后端时使用。
State backends can be configured per job. In addition, you can define a default state backend to be used when the
job does not explicitly define a state backend.

### 为每一个job配置状态后端
### Setting the Per-job State Backend

每个job的状态后端可以在`StreamExecutionEnvironment`中设置，就像下面展示的例子：
The per-job state backend is set on the `StreamExecutionEnvironment` of the job, as shown in the example below:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.setStateBackend(new FsStateBackend("hdfs://namenode:40010/flink/checkpoints"))
{% endhighlight %}
</div>
</div>


### 设置默认的状态后端
### Setting Default State Backend

一个默认的状态后端可以在`flink-conf.yaml`配置文件中使用`state.backend`配置关键字来配置。
A default state backend can be configured in the `flink-conf.yaml`, using the configuration key `state.backend`.

可选的值是*jobmanager* (MemoryStateBackend), *filesystem* (FsStateBackend),或者一个实现了状态后端工厂[FsStateBackendFactory](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/runtime/state/filesystem/FsStateBackendFactory.java)的完全限定类名，例如`org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory` 代表RocksDBStateBackend.
Possible values for the config entry are *jobmanager* (MemoryStateBackend), *filesystem* (FsStateBackend), or the fully qualified class
name of the class that implements the state backend factory [FsStateBackendFactory](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/runtime/state/filesystem/FsStateBackendFactory.java),
such as `org.apache.flink.contrib.streaming.state.RocksDBStateBackendFactory` for RocksDBStateBackend.

在默认的状态设置为*filesystem*的情况下，入口`state.backend.fs.checkpointdir` 定义检查点数据将被保存的目录。
In the case where the default state backend is set to *filesystem*, the entry `state.backend.fs.checkpointdir` defines the directory where the checkpoint data will be stored.

配置文件中一段配置可以像下面这样：

A sample section in the configuration file could look as follows:

~~~
# The backend that will be used to store operator state checkpoints

state.backend: filesystem


# Directory for storing checkpoints

state.backend.fs.checkpointdir: hdfs://namenode:40010/flink/checkpoints
~~~
