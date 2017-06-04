---
title: "Debugging and Tuning Checkpoints and Large State"
nav-parent_id: monitoring
nav-pos: 12
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

本页面针对大型应用场景下的配置和调优提供指导
* ToC
{:toc}

## 概述


要保证Flink应用程序在大型应用中的可靠性，两个条件必须要满足：

  - 应用需要能可靠地获取检查点(checkpoints)

  - 资源需要充足，当出现错误后能及时跟上输入数据流

第一部分讨论了规模应用下如何得到高性能的检查点。
最后部分分析了一些最佳实践，关注于资源使用规划。


## 监控状态和检查点

监控检查点行为的最简单的方法，是通过UI的检查点部分(checkpoint section)。文档[检查点监控](checkpoint_monitoring.html)
展示了如何获取有效的检查点参数

当检查点数量规模在不多扩大时，有两个数字需要特别关注：

  - 算子开始处理它们的检查点的延迟时间：这个时间目前没有直接暴露出来，但是这个时间等同于：
    
    `checkpoint_start_delay = end_to_end_duration - synchronous_duration - asynchronous_duration`

    当触发检查点的时间值维持很高的状态时，就意味着 *checkpoint barriers* 需要花费较长时间从数据源传输到算子。这通常表明当前系统正运行在
    持续的压力下。

  - 数据排列过程中的数据缓冲量。对于exactly-once 语义来说，Flink 在接收多个输入数据流的算子中进行 *aligns* 数据流的同时，会为排列操作缓存一些数据。这个缓存数据容量理论上时很低的 —— 更大的数量意味着从不同的输入流中接收到的checkpoint barriers 
  的时间差异很大。

注意，在临时性的系统压力，数据倾斜或网络故障情况下，此处提到的数字会表现出临时性增高。但是，如果这些数字一直都很高，那就意味着Flink将太多的资源放到checkpointing了。

## 调整 Checkpointing

检查点通常在应用程序可配置的时间区间内被触发。当完成一个检查点所花费的时间超过了检查点的时间间隔，那么下一个检查点在当前检查点处理过程尚未完成前不会被触发。默认情况下，当前正在处理的检查点一旦结束，下一个检查点会立即被触发。

当检查点结束时间超过基本的时间区间的情况变得很频繁时（比如由于数据规模增长超过预期，或保存检查点的存储突然变慢），系统会持续地获取检查点（一旦处于执行状态的检查点结束，新的检查点立即启动）。这也就意味着太多的资源被持续的绑定到了checkpointing中，并且运算进展缓慢。这种情况对使用异步检查状态来处理流式数据的应用产生的影响较小，但还是可能对整个应用的效率造成影响。

为了避免这种情况发生，应用需要定义一个*minimum duration between checkpoints*（检查点之间的最小时间间隔）:

`StreamExecutionEnvironment.getCheckpointConfig().setMinPauseBetweenCheckpoints(milliseconds)`

这个间隔时间是是一个最小的时间区间，时间跨度是从上一个检查点结束到下一个检查点开始。下图说明了这个间隔时间是如何影响checkpointing的。

<img src="../fig/checkpoint_tuning.svg" class="center" width="80%" alt="Illustration how the minimum-time-between-checkpoints parameter affects checkpointing behavior."/>

*注意:* 应用可进行配置（通过`CheckpointConfig`）以允许多个检查点同时被处理。对于使用Flink的大型应用，通常会绑定太多的资源到
checkpointing中。当一个保存点被手动触发，它将跟正在处理过程中的检查点同时处理。

## 调整 网络缓存（Network Buffers）

在大型应用中，网络缓存的数量能很容易地影响checkpointing。Flink社区正致力于在下一个Flink版本中努力消除这个参数。

网络缓存的数量定义了一个TaskManager运行过程中，在被背压压垮前，能容纳多少in-flight数据量。
一个非常高的网络缓存意味着当检查点启动时，有大量数据存储于流式网络通道中。由于checkpoint barriers会带着这些数据传输(参见 [description of how checkpointing works](../internals/stream_checkpointing.html))，大量的in-flight数据意味着barriers必须等待这些数据在到达目标算子前先被传输/处理完毕。

拥有大量的in-flight数据在整体上并不能提高数据处理速度。它仅意味着从数据源（日志，文件，消息队列）获取数据会更快，并且在Flink中被缓存的更久。拥有较少的网络缓存意味着
在数据被真正处理前，我们更直接地从数据源获取数据，这通常也是我们希望的。
因此，网络缓存数量不应该被设置为任意大，而应该设置成所需缓存数最小值的低倍数（比如2倍）。


## 尽可能的将状态检查设置为异步操作

当状态是*asynchronously*（异步）快照的，检查点的计算效率会比*synchronously*（同步）快照的更高。特别是在包含了多个join操作、Co-functions或window操作的复杂流应用中，会产生很大的影响。

为了获取异步快照的状态，应用需要做两件事情：

  1. 使用[managed by Flink](../dev/stream/state.html)的状态:托管的状态意味着Flink提供状态存储时的数据结构。当前，*keyed state*（包含键值的状态）是这样的，这类状态是对相关接口的抽象，比如`ValueState`, `ListState`, `ReducingState`, ...

  2. 使用支持异步快照的状态后端。在Flink 1.2版本，只有RocksDB状态后端使用了完全的异步快照。

上述两点表明，（在Flink 1.2版本中）大数据量的状态通常应该以基于键值的状态存储，而不是运算状态。
这将随着对 *managed operator state* （运算状态托管）的引入计划而改变。

## 调整 RocksDB

很多大规模Flink流处理应用的状态存储器，都使用了*RocksDB State Backend*。这个后端的运行效果已经超过了主内存，并且能可靠存储大规模的[keyed state](../dev/stream/state.html)。

但糟糕的是，RocksDB的运行效果会随配置不同而变化，而且几乎没有文档说明如何合适的调整RockDB的配置。比如，默认的配置是针对固态硬盘而设置的，在旋转磁盘上的运行效果不佳。

**Passing Options to RocksDB**

{% highlight java %}
RocksDBStateBackend.setOptions(new MyOptions());

public class MyOptions implements OptionsFactory {

    @Override
    public DBOptions createDBOptions() {
        return new DBOptions()
            .setIncreaseParallelism(4)
            .setUseFsync(false)
            .setDisableDataSync(true);
    }

    @Override
    public ColumnFamilyOptions createColumnOptions() {

        return new ColumnFamilyOptions()
            .setTableFormatConfig(
                new BlockBasedTableConfig()
                    .setBlockCacheSize(256 * 1024 * 1024)  // 256 MB
                    .setBlockSize(128 * 1024));            // 128 KB
    }
}
{% endhighlight %}

**预定义的 配置项**

Flink为RocksDB的不同设置提供了一些预定义的配置项集合，这些配置项集合的配置方式如：
`RocksDBStateBacked.setPredefinedOptions(PredefinedOptions.SPINNING_DISK_OPTIMIZED_HIGH_MEM)`.

我们希望逐渐积累更多的这类配置。当你发现一组配置项很好用，并且针对某些工作场景具有代表性，那么我们希望你能贡献这些预定义的配置项。

**重要说明** RocksDB是一个本地的库，它分配的内存不是来自于JVM，而是直接来自于进程的本地内存。任何你分配给RocksDB 的内存都需要被算到进程的本地内存里，通常是通过减少相同数量的TaskManagers的JVM堆内存来实现。不要去分配比配置里面更多的内存，这会导致YARN/Mesos/等终止JVM 进程。


## 性能规划

这部分探讨了如何决定每个Flink job该使用多少资源，以保证稳定运行。
性能规划的首要基本规则如下：

  - 普通的运算需要拥有足够的容量，以避免运算长期运行在*back pressure* 下。
    参考[back pressure monitoring](back_pressure.html)，了解更多关于如何检测应用是否工作在back pressure下。

  - 在无故障期间，在应用无压力运行时所需的最大资源的基础上，准备一些额外的资源。
    这些资源在应用恢复期间及时处理掉堆积的输入数据时很有必要。
    至于需要准备多少资源，取决于恢复计算通常花费多少时间（这个时间一般取决于有多少状态数据需要加载到新的TaskManager里），以及方案要求的故障恢复时间。

    *重要说明*: 基线应该在checkpointing 处于启动状态时发布，因为checkpointing 会绑定一定数量的资源（比如网络带宽）。

  - 暂时出现的运行压力通常是不会有问题的，这是负载高峰期、catch-up阶段，或外部系统（即在一个sink里进行写入的系统）表现出临时性降速时的一个重要部分。

  - 某些操作（比如大数据量的window操作）会导致其下游操作出现负载峰值：
    在出现这类window操作的情况下，当这类window被创建时，其下游操作几乎没有什么事做，而当这类窗口被放出时，下游操作会遇到一个负载峰值。
    在规划下游操作的并行方案时，需要考虑到这类窗口放出的数量，以及这类负载峰值需要以多快的速度被处理掉。

**重要说明** 为了能支持后续资源扩展，需要确保为数据流程序设置了合理的*maximum parallelism* （最大并发数参数）。最大并发数参数定义了在re-scaling应用程序时（通过一个savepoint 保存点），我们能设置多高的并发度。

Flink的内部记录系统会以max-parallelism-many *key groups* 的粒度跟踪并发状态。
Flink的设计力求在设置了很大的最大并发度参数时仍有效率，即使是以一个低并发度执行程序。



