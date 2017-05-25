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

Flink的web界面提供了一个标签去监控作业的检查点.统计的结果在作业结束之后也始终可见.用四种不同的标签来展示检查点的相关信息:概述(Overview),历史（History）,摘要(Summary)和配置（Configuration）.下面的章节将按顺序覆盖这些信息.

## 监控

### 概述标签

概述标签列举了以下统计信息。请注意这些统计信息将在作业管理器丢失后失效，并且如果作业管理器发生故障转移统计信息也将会被重置。

- **检测点统计**
	- 已触发: 自作业启动开始已经触发的检查点总量.
	- 进行中: 当前正在进行的检查点数量.
	- 已完成: 自作业启动开始已经成功的检查点总量.
	- 已失败: 自作业启动开始已经失败的检查点总量.
	- 已恢复: 从作业启动开始恢复操作的次数.这也是告诉你提交开始作业已经重启的次数.请注意，使用保存点的初始提交也将作为还原计数，并且如果JobManager在操作过程中丢失，则重置计数。
- **最近完成的检查点**: 最近成功完成的检查点。点击`更多详情`将会给你提供子任务级别的详细统计信息.
- **最近失败的检查点**:最近失败的检查点. 点击`更多详情`将会给你提供到子任务级别的详细统计信息.
- **最近的保存点**: 利用外部路径存储最新触发的保存点.点击`更多详情`将会给你提供到子任务级别的详细统计信息.
- **最新的恢复**: 有两种类型的恢复操作.
	- 来自检查点的恢复：从定期检查点进行恢复.
	- 来自保存点的恢复：从保存点进行恢复

### 历史标签

检查点历史记录了最近触发的和进行中的检查点相关统计信息.

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-history.png" width="700px" alt="检查点监控: 历史">
</center>

- **ID**: 触发的检查点的ID.每个检测点的ID是递增的,从1开始.
- **状态**: 检查点的当前状态，它可能是 *进行中* (<i aria-hidden="true" class="fa fa-circle-o-notch fa-spin fa-fw"/>), *已完成* (<i aria-hidden="true" class="fa fa-check"/>), 或者*已失败* (<i aria-hidden="true" class="fa fa-remove"/>). 如果触发的检查点是一个保存点，你将会看到一个 <i aria-hidden="true" class="fa fa-floppy-o"/> 符号.
- **触发时间**: 在作业管理器上触发的检查点的时间.
- **最近确认**: 在作业管理器上收到的任何子任务的最新确认时间（或如果尚未收到确认，则为n/a）.
- **端到端持续时间**: 从触发时间戳到最新确认的持续时间（或如果没有接收到确认，则为n/a）.对应一个完整的检查点的端到端持续时间，由确认检查点的最后一个子任务所确定，这个时间通常大于单个子任务实际需要去检查状态的时间.
- **状态大小**: 所有已确认状态的子任务大小
- **缓冲持续对齐**:在所有确认的子任务对齐期间缓冲的字节数。如果在校验点期间发生流对齐，则此值仅为0。如果检查点模式是`最少一次（AT_LEAST_ONCE）`，则始终为0，因为这种模式不需要流对齐.

#### 历史大小配置

可以通过以下配置关键字来配置记录的最近检查点的数量.默认是`10`.
```sh
# 记录最近的检查点的数量
jobmanager.web.checkpoints.history: 15
```

### 摘要标签

摘要计算了端到端间隔内所有已完成检查点的一个简单的最小/平均/最大统计信息,状态大小和对齐期间缓冲的字节(有关这些含义的详细信息，请参阅[历史记录](#history)).

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-summary.png" width="700px" alt="检查点监控: 摘要">
</center>

请注意这些统计信息将在作业管理器丢失后失效，并且如果作业管理器发生故障转移统计信息也将会被重置。


### 配置标签

以下配置列举了您的流（streaming）配置信息：

- **检查点模式**:  是*恰好一次（Exactly Once）* 还是 *最少一次（AT_LEAST_ONCE）*.
- **间隔**: 配置检查点间隔.在这个间隔触发检查点
- **超时**: 超时之后，作业管理器将取消一个检查点，并触发一个新的检查点.
- **检查点之间的最小暂停时间**: 检查点之间的最短停留时间。在检查点成功完成后，我们至少需要等待这段时间才会触发下一个检查点，潜在的延迟定期执行间隔.
- **检查点之间的最大暂停时间**: 可以同时进行的最大检查点数.
- **外部持久化检查点**: 启用或禁用。如果启用，还列出了外部检查点的清理配置（取消时删除或保留）.

### 检查点详情

当您点击一个检查点的*更多详情*链接时，您将获得一个所有算子的最小/平均/最大值摘要值，以及每个子任务的详细个数

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details.png" width="700px" alt="检查点监控: 详情">
</center>

#### 每个算子的摘要

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details_summary.png" width="700px" alt="检查点监控: 详情摘要">
</center>

#### 所有任务统计

<center>
  <img src="{{ site.baseurl }}/fig/checkpoint_monitoring-details_subtasks.png" width="700px" alt="检查点监控: 子任务">
</center>
