---
title: "Checkpointing"
nav-parent_id: streaming
nav-pos: 50
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

Flink中的每个function和operator都可以是**有状态的**（有关详细信息请参阅[working with state](state.html)）。有状态的functions通过处理各个元素/事件来存储数据，并把状态作为关键构建以支持任何类型更复杂的操作。

为了使状态能够容错，Flink需要状态的**检查点**。Flink通过检查点恢复流中的状态和位置，进而使得应用程序与无故障执行具有相同的语义。

[documentation on streaming fault tolerance](../../internals/stream_checkpointing.html) 详细介绍了Flink流容错机制的技术。


## 先决条件

Flink的检查点机制与流和状态的持久化存储交互，一般来说该机制需要：

  - *持久化的*数据source，它可以在一定时间内重放事件。这种数据sources的典型例子是持久化的消息队列（比如Apache Kafka，RabbitMQ，Amazon Kinesis，Google PubSub）或文件系统（比如HDFS，S3，GFS，NFS，Ceph，。。。）。
  - 用于状态的持久化存储器，通常是分布式文件系统（比如HDFS，S3，GFS，NFS，Ceph，。。。）


## 启用和配置检查点

默认情况下，检查点被禁用。要启用检查点，请在`StreamExecutionEnvironment`上调用`enableCheckpointing(n)`方法，其中*n*是以毫秒为单位的检查点间隔。

检查点的其他参数包括：

  - *exactly-once vs. at-least-once*：你可以从这两种模式中选择一种模式传递给`enableCheckpointing(n)`方法。Exactly-once对于大多数应用来说是最合适的。At-least-once可能用在某些延迟超低的应用程序（始终延迟为几毫秒）。

  - *检查点超时*：如果检查点构造时间超过该值，则终止正在构建的检查点。

  - *检查点间的最短时间*：为了确保流应用程序在检查点之间能有一定程度的进展，可以设定检查点之间最短的时间。如果该值设置为*5000*，则下一个检查点将在上一个检查点完成后5秒钟内启动，而不管检查点持续时间和检查点间隔。注意这意味着检查点间隔参数应该永远不小于此参数。
    
    通过先定义*检查点间的最短时间*，再定义检查点间隔，可以更容易地配置应用程序，因为“检查点间的最短时间”不容易受到检查点有时耗时比平均更长的事实的影响（例如，如果存放检查点的目标存储系统暂时缓慢）。

    注意该值还意味着并发的检查点数为*1*。

  - *并发的检查点数*：默认情况下，系统不会在进行一个检查点时再触发另一个检查点。这能确保拓扑不会因为在检查点上耗时过多以致流处理进展缓慢。有些情况允许多个重叠的检查点是有意义的：对于有固定处理延时的pipelines（比如因为函数调用外部服务而需要一些响应时间），但仍需要做非常频繁的检查点（100毫秒）以减轻遇到错误时重新处理的代价。

    当定义了“检查点间的最短时间”，就不能使用此选项。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000)

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig.setMinPauseBetweenCheckpoints(500)

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig.setCheckpointTimeout(60000)

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
{% endhighlight %}
</div>
</div>

{% top %}


## 选择状态的后端存储（State Backend）

检查点机制将数据source和数据sink的进度，window的状态以及[用户定义状态](state.html)一致地存储起来以提供*exactly once*语义。存储检查点的位置（例如，JobManager的内存，文件系统，数据库）取决于配置的**State Backend**。

默认情况下，状态将保存在内存中，检查点将存储在主节点（JobManager）的内存中。 为了正确地保留大状态，Flink支持各种形式的存储和检查点状态，可以通过`StreamExecutionEnvironment.setStateBackend(…)`进行设置。

参阅 [state backends](../../ops/state_backends.html) 了解更多关于支持的state backends以及作业端和集群端的详细配置。


## 迭代式作业中的状态检查点（State Checkpoints in Iterative Jobs）

Flink目前只为非迭代式作业提供处理保证。在迭代式作业上启用检查点会抛出异常。要想在迭代式作业上强启检查点，你需要在启用检查点时设置特殊标识：`env.enableCheckpointing(interval, force = true)`。

请注意在故障期间正在循环迭代（loop edges）中的记录（以及与之相关的状态更改）将会丢失。

{% top %}


## 重启策略（Restart Strategies）

Flink支持不同的重启策略，以在故障发生时控制作业如何重启。更多详情请参阅 [Restart Strategies]({{ site.baseurl }}/dev/restart_strategies.html)。

{% top %}

