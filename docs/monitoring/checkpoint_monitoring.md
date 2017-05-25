---
title: "Monitoring Checkpointing"
nav-parent_id: monitoring
nav-pos: 4
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

## 概述

Fink的web节目提供了一个标签去监控作业的监测点.这些统计结果在作业结束之后也始终可见.这里有四种不同的标签来展示监测点的想信息:概要(Overview),历史（History）,摘要(Summary)和配置（Configuration）.下面的章节将按顺序覆盖这些.

## 监控

### 概要标签

概述选项卡列出以下统计信息。请注意，这些统计信息不能在JobManager丢失后生效，并且如果您的JobManager故障转移重置。

- **检测点统计**
	- 已触发: 自作业启动开始已经触发的检查点总量.
	- 进行中: 当前正在进行的检测点数量.
	- 已完成: 自作业启动开始已经成功的检查点总量.
	- 已失败: 自作业启动开始已经失败的检查点总量.
	- 已恢复: The number of restore operations since the job started. This also tells you how many times the job has restarted since submission. Note that the initial submission with a savepoint also counts as a restore and the count is reset if the JobManager was lost during operation.
- **Latest Completed Checkpoint**: The latest successfully completed checkpoints. Clicking on `More details` gives you detailed statistics down to the subtask level.
- **Latest Failed Checkpoint**: The latest failed checkpoint. Clicking on `More details` gives you detailed statistics down to the subtask level.
- **最新的检测点**: The latest triggered savepoint with its external path. Clicking on `More details` gives you detailed statistics down to the subtask level.
- **最新的恢复**: 有两种类型的恢复操作.
	- 来自检测点的恢复：从一个常规的周期性检测点进行恢复.
	- 来自保存点的恢复：从一个保存点进行恢复

### 历史标签

检测点历史记录了最近触发的和进行中的检测点统计数据.

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-history.png" width="700px" alt="Checkpoint Monitoring: History">
</center>

- **ID**: 触发检测点的ID.每个检测点的ID是递增的,从1开始.
- **状态**:  检测点的当前状态，它可能是 *进行中* (<i aria-hidden="true" class="fa fa-circle-o-notch fa-spin fa-fw"/>), *已完成* (<i aria-hidden="true" class="fa fa-check"/>), 或者*已失败* (<i aria-hidden="true" class="fa fa-remove"/>). 如果触发检测点是一个保存点，你将会看到一个 <i aria-hidden="true" class="fa fa-floppy-o"/> 符号.
- **触发时间**:  在作业管理器上触发检测点的时间.
- **最新确认**: 在JobManager收到任何子任务的最新确认的时间（或如果尚未收到确认，则为n/a）.
- **端到端间隔**: 从触发时间戳到最新确认的持续时间（或如果没有接收到确认，则为n/a）.对应一个完整的检测点的端到端间隔时间，由确认检测点的最后一个子任务所确定，这个时间通常大于单个子任务需要去实际检查状态的时间.
- **状态大小**:  所有已确认状态的子任务大小
- **对其缓存间隔**:在所有确认的子任务对齐期间缓冲的字节数。如果在校验点期间发生流对齐，则此值仅为0。如果检查点模式为`AT_LEAST_ONCE`，则始终为0，因为至少一次模式不需要流对齐

#### 历史大小配置

可以通过以下配置关键字来配置记录的最近检测点的数量.默认是`10`.
```sh
# Number of recent checkpoints that are remembered
jobmanager.web.checkpoints.history: 15
```

### 摘要标签

摘要计算了端到端间隔内所有已完成检测点的一个简单的最小/平均/最大统计,统计大小和间隔对其字节缓存大小(通过查看 [历史](#history) 来了解这些含义的细节).

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-summary.png" width="700px" alt="Checkpoint Monitoring: Summary">
</center>

注意这些统计不能再JobManager丢失后生效，并且如果你的JobManager故障转移重置.

### 配置标签

这些配置列举了流（streaming）配置：

- **检测点模式**:  是*Exactly Once* 还是 *At least Once*.
- **间隔**: 配置检测点间隔.在这个间隔触发检测点
- **超时**: 超时之后，JobManager将取消一个检查点，并触发一个新的检查点.
- **两个检测点的最小暂停时间**: 检查点之间的最短停留时间。在检查点成功完成后，我们至少等待这段时间才触发下一个时间，可能延迟定期间隔。
- **两个检测点的最大暂停时间**: 可以同时进行的最大检查点数.
- **外部持久化检测点**: 启用或禁用。如果启用，还列出了外部检查点的清理配置（取消时删除或保留）.

### 检查点详情

当你点击一个检测点的*更多详情信息*链接时，可以获得一个所有算子的Minumum/Average/Maximum摘要，以及每个子任务的详细个数

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details.png" width="700px" alt="Checkpoint Monitoring: Details">
</center>

#### 每个算子摘要

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details_summary.png" width="700px" alt="Checkpoint Monitoring: Details Summary">
</center>

#### 所有任务统计

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details_subtasks.png" width="700px" alt="Checkpoint Monitoring: Subtasks">
</center>
