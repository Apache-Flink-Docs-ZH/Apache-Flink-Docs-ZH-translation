---
title: "生产环境检查清单"
nav-parent_id: setup
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

* ToC
{:toc}

## 生产环境检查清单

生产环境检查清单的目的是，当发布Flink任务到**生产环境**时，清单提供简要概述一些重要且需要**仔细考虑**配置项。 
大多数参数Flink在安装时提供了的默认设置，使Flink的使用和采用更加容易。 
对于大多数用户和场景，那些默认参数是一个好的起点，完全适用于一次性的任务。 

然而，一旦计划将Flink应用程序发布到生产环境，对配置的要求就会提高。 
例如，如果需要（重新）扩展或升级Flink任务，又或者升级Flink的版本。 

下文中，我们将介绍一系列配置选项，发布到生产环境前，必须先检查这些配置选项。

### 设置 operator 的最大并行度

最大并行度是Flink 1.2中新引入的配置参数，对Flink任务（重新）扩展性具有重要的意义。
该参数可以对每个job和/或每个 operator 的粒度来设置您可以扩展operator的最大并行度。
需要知道的是一旦任务运行后，将**没有办法改变**这个参数，除非完全重新启动你的任务（比如重一个新状态开始，而不是来自先前的checkpoint/savepoint）。

即使Flink将来可能提供一些方法对现有savepoints改变的最大并行性，但是你需要知道的是这将是你需要避免的长运行操作。
在这一点上，您可能会想知道为什么不给该参数一个非常高的值的默认值。
这样做的原因是高最大并行度可以对你有一些影响应用程序的性能甚至状态大小，
因为Flink必须维护一定元数据来保证它有重新伸缩的能力，从而可以提升最大的平行度增加。

一般来说，你应该选择一个最大的并行度足以保证你的未来的可扩展性需求，
但保持尽可能低的值可以提供略微更好的性能。尤其是，最大并行度高于128将通常导致 keyed backends 产生稍微更大的状态快照。

请注意，最大并行度必须满足以下条件： 

`0 < 并行度  <= 最大并行度 <= 2^15`

可以通过 `setMaxParallelism（int maxparallelism)` 设置最大并行度。 
默认情况下，Flink任务首次启动时会选择最大并行度作为并行性的函数: 

- `128` : 并行度 小于128时.
- `MIN(nextPowerOfTwo(并行度 + (并行度 / 2)), 2^15)` : 并行度 大于128时.

### 给 operators 设置 UUID

如[savepoints]（{{site.baseurl}} / setup / savepoints.html）的文档中所述，
用户应该为给 operator 设置 uid。这些operator 的 uid 对 flink 中 operator states 和 operators 对应关系很重要，
这点反过来对于savepoints是至关重要的。默认情况下，operator 的 uid 通过遍历JobGraph和Hash 特定operator 的属性来生成的。
虽然从用户的角度来看，这是很舒适的，但它也非常脆弱，因为对JobGraph的更改（例如，交换operator）将导致产生新的UUID。 
为了建立稳定的映射，我们需要用户来通过 `setUid（String uid）` 来设定稳定的operator uid。 

### 选择state backend
目前，Flink有一个局限性，它只能从保存点的同一状态后端 恢复到的savepoint的state。 
例如，这意味着我们无法使用内存的state backend做savepoint，因此任务更改为使用RocksDB tate backend并进行还原。 
虽然我们计划在不久的将来使 backend 间可以交互，但现在还没有实现。 这也意味着在生产环境中，应该仔细考虑给你的任务使用对应的backend。 

一般来说，我们建议使用RocksDB，因为这是目前唯一的一种state backend能够支持大状态(state)（例如，状态（state）超过可用主内存）和异步快照。 根据我们的经验，异步快照对于大型状态（state）非常重要，因为它们不会阻塞 operator ，Flink可以在不停止流处理的情况下写入快照（snapshot）。 然而，RocksDB的性能可能会比例如基于内存的 state backend 要差。 如果能保证你的状态（state）永远不会超过主内存，也不阻碍数据流的写入，它就不是问题，
你**可以考虑**来使用RocksDB backend。 但是，在这一点上，我们**强烈建议**生产环境中使用RocksDB。
