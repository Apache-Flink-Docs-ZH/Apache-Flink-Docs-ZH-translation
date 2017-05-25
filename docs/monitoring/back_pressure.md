---
title: "Monitoring Back Pressure"
nav-parent_id: monitoring
nav-pos: 5
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

Flink的web界面提供了一个选项去监控正在运行任务的反压行为.

* ToC
{:toc}

## 反压

对于一个任务如果你看到了一个**反压警告**(例如. `High`),这意味着正在生产的数据比下游算子消耗的数据快。任务中的记录流向下游（例如：从源头到落地），反压则在流上沿相反方向传播，向上流动.

举一个简单的`源(source)`端到`落地（sink）`端作业为例，如果你在`源(source)`端看到一个警告，这意味着`落地（sink）`端消费数据的速度慢于`源(source)`端正在生产的速度。`落地（sink）`端开始反压上游算子源

 
## 抽样线程

反压监测工作，是通过反复跟踪正在运行任务的堆栈样本来实现，JobManager触发器重复调用作业任务的`Thread.getStackTrace()`方法

<img src="{{ site.baseurl }}/fig/back_pressure_sampling.png" class="img-responsive">
<!-- https://docs.google.com/drawings/d/1_YDYGdUwGUck5zeLxJ5Z5jqhpMzqRz70JxKnrrJUltA/edit?usp=sharing -->

如果样本显示一个任务线程阻塞（Stuck）在某个内部方法调用上（从网络堆栈请求缓冲区），这表明该任务有反压.

默认情况下，JobManager会每隔50ms触发对一个作业的每个任务依次进行100次堆栈跟踪调用,根据调用结果来确认反压。在web界面看到的比率值，它表示在一个内部方法调用中阻塞（Stuck）的堆栈跟踪次数。例如:`0.01`表明100次仅有1次方法调用被阻塞

- **OK**: 0 <= Ratio <= 0.10
- **LOW**: 0.10 < Ratio <= 0.5
- **HIGH**: 0.5 < Ratio <= 1

为了防止JobManager堆栈跟踪抽样超负荷，web界面刷新抽样频率为60秒

## 配置

可以通过以下参数来配置JobManager的抽样次数：

- `jobmanager.web.backpressure.refresh-interval`: 表示采样统计结果刷新时间间隔(默认: 60000, 1 分钟).
- `jobmanager.web.backpressure.num-samples`: 确定反压状态，所使用的堆栈跟踪调用次数(默认: 100).
- `jobmanager.web.backpressure.delay-between-samples`: 确定反压状态，表示对一个作业的每个任务依次调用堆栈跟踪的时间间隔 (DEFAULT: 50, 50 ms).



## 示例

可以在Job Overview后面发现*Back Pressure*

### 采用进行中


这表明JobManager触发了一个堆栈跟踪抽样运行任务.使用默认配置，任务大约5秒钟完成

注意点击这一行，将会触发这个算子所有子任务的抽样

<img src="{{ site.baseurl }}/fig/back_pressure_sampling_in_progress.png" class="img-responsive">

### 反压状态

如果看到这些任务的状态为**OK**，表明没有反压，如果是**High**，那就意味着这些任务有反压

<img src="{{ site.baseurl }}/fig/back_pressure_sampling_ok.png" class="img-responsive">

<img src="{{ site.baseurl }}/fig/back_pressure_sampling_high.png" class="img-responsive">
