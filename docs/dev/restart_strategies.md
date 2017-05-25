---
title: "Restart Strategies"
nav-parent_id: execution
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

Flink 支持多种不同的重启策略，这些策略控制了在失败情况下工作要如何重启。
集群在启动时会伴随一个默认的重启策略，在没有定义具体工作重启策略时会使用该默认策略。
如果在工作提交时制定一个重启策略，该策略会覆盖集群的默认设定。

* This will be replaced by the TOC
{:toc}

## 概览

默认的重启策略可以通过 Flink 的配置文件 `flink-conf.yaml` 指定。
配置参数 *restart-strategy* 定义了哪个策略被使用。
如果没有启用 checkpointing，则使用无重启 (no restart) 策略。
如果启用了 checkpointing，但没有配置重启策略，则使用固定间隔 (fixed-delay) 策略，其中 `Integer.MAX_VALUE` 参数是尝试重启次数。
参阅下列可用的重启策略来了解什么值能被支持。

每个重启策略都有自己的一组参数来控制策略的行为。
这些值也可以在配置文件中设置。
每个重启策略的描述包括了更多关于对应配置值的信息。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 50%">重启策略</th>
      <th class="text-left">对应值</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>固定间隔 (Fixed delay)</td>
        <td>fixed-delay</td>
    </tr>
    <tr>
        <td>失败率 (Failure rate)</td>
        <td>failure-rate</td>
    </tr>
    <tr>
        <td>无重启 (No restart)</td>
        <td>none</td>
    </tr>
  </tbody>
</table>

除了定义默认的重启策略，也可以为每个 Flink 工作定义一个具体的重启策略。
这个重启策略可以通过调用 `ExecutionEnvironment` 的 `setRestartStrategy` 方法在编程时设置。
该方法对 `StreamExecutionEnvironment` 同样有效。

下列例子展示我们如何为我们的工作设置一个固定间隔重启策略。
在失败的情况下，系统会重启工作 3 次，并在连续两次尝试重启中等待 10 秒。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // 尝试重启的次数
  Time.of(10, TimeUnit.SECONDS) // 间隔
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // 尝试重启的次数
  Time.of(10, TimeUnit.SECONDS) // 间隔
))
{% endhighlight %}
</div>
</div>

{% top %}

## 重启策略

以下各部分描述与具体重启策略相关的配置选项。

### 固定间隔 (Fixed Delay) 重启策略

The fixed delay restart strategy attempts a given number of times to restart the job.
If the maximum number of attempts is exceeded, the job eventually fails.
In-between two consecutive restart attempts, the restart strategy waits a fixed amount of time.

This strategy is enabled as default by setting the following configuration parameter in `flink-conf.yaml`.

~~~
restart-strategy: fixed-delay
~~~

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Configuration Parameter</th>
      <th class="text-left" style="width: 40%">Description</th>
      <th class="text-left">Default Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><code>restart-strategy.fixed-delay.attempts</code></td>
        <td>The number of times that Flink retries the execution before the job is declared as failed.</td>
        <td>1, or <code>Integer.MAX_VALUE</code> if activated by checkpointing</td>
    </tr>
    <tr>
        <td><code>restart-strategy.fixed-delay.delay</code></td>
        <td>Delaying the retry means that after a failed execution, the re-execution does not start immediately, but only after a certain delay. Delaying the retries can be helpful when the program interacts with external systems where for example connections or pending transactions should reach a timeout before re-execution is attempted.</td>
        <td><code>akka.ask.timeout</code>, or 10s if activated by checkpointing</td>
    </tr>
  </tbody>
</table>

For example:

~~~
restart-strategy.fixed-delay.attempts: 3
restart-strategy.fixed-delay.delay: 10 s
~~~

The fixed delay restart strategy can also be set programmatically:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
  3, // number of restart attempts
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### 失败率 (Failure Rate) 重启策略

The failure rate restart strategy restarts job after failure, but when `failure rate` (failures per time interval) is exceeded, the job eventually fails.
In-between two consecutive restart attempts, the restart strategy waits a fixed amount of time.

This strategy is enabled as default by setting the following configuration parameter in `flink-conf.yaml`.

~~~
restart-strategy: failure-rate
~~~

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 40%">Configuration Parameter</th>
      <th class="text-left" style="width: 40%">Description</th>
      <th class="text-left">Default Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td><it>restart-strategy.failure-rate.max-failures-per-interval</it></td>
        <td>Maximum number of restarts in given time interval before failing a job</td>
        <td>1</td>
    </tr>
    <tr>
        <td><it>restart-strategy.failure-rate.failure-rate-interval</it></td>
        <td>Time interval for measuring failure rate.</td>
        <td>1 minute</td>
    </tr>
    <tr>
        <td><it>restart-strategy.failure-rate.delay</it></td>
        <td>Delay between two consecutive restart attempts</td>
        <td><it>akka.ask.timeout</it></td>
    </tr>
  </tbody>
</table>

~~~
restart-strategy.failure-rate.max-failures-per-interval: 3
restart-strategy.failure-rate.failure-rate-interval: 5 min
restart-strategy.failure-rate.delay: 10 s
~~~

The failure rate restart strategy can also be set programmatically:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per interval
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.failureRateRestart(
  3, // max failures per unit
  Time.of(5, TimeUnit.MINUTES), //time interval for measuring failure rate
  Time.of(10, TimeUnit.SECONDS) // delay
))
{% endhighlight %}
</div>
</div>

{% top %}

### 无重启 (No Restart) 策略

The job fails directly and no restart is attempted.

~~~
restart-strategy: none
~~~

The no restart strategy can also be set programmatically:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setRestartStrategy(RestartStrategies.noRestart());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()
env.setRestartStrategy(RestartStrategies.noRestart())
{% endhighlight %}
</div>
</div>

### 回调 (Fallback) 重启策略

The cluster defined restart strategy is used. 
This helpful for streaming programs which enable checkpointing.
Per default, a fixed delay restart strategy is chosen if there is no other restart strategy defined.

{% top %}
