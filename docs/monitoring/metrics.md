---
title: "Metrics"
nav-parent_id: monitoring
nav-pos: 1
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

Flink公开了一个指标系统，可以收集和暴露指标给外部系统.
* This will be replaced by the TOC
{:toc}

## Registering metrics
你可以调用 `getRuntimeContext().getMetricGroup()`方法来访问任何继承自[RichFunction]({{ site.baseurl }}/dev/api_concepts.html#rich-functions)函数的用户函数的指标系统.这个方法返回一个`MetricGroup`对象，通过这个对象可以创建和注册新的指标.

### Metric types

Flink支持的指标类型：`Counters`,`Gauges`,`Histograms`和`Meters`.

#### Counter
`Counter`用作某方面计数，通过调用`inc()/inc(long n) `或者 `dec()/dec(long n)`方法来使当前的值增加或者减少. 
 通过调用`MetricGroup`的`counter(String name)`方法可以创建和注册一个`Counter`.

{% highlight java %}

public class MyMapper extends RichMapFunction<String, Integer> {
  private Counter counter;

  @Override
  public void open(Configuration config) {
    this.counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCounter");
  }

  @public Integer map(String value) throws Exception {
    this.counter.inc();
  }
}

{% endhighlight %}

或者你也可以使用自己实现的`Counter`：

{% highlight java %}

public class MyMapper extends RichMapFunction<String, Integer> {
  private Counter counter;

  @Override
  public void open(Configuration config) {
    this.counter = getRuntimeContext()
      .getMetricGroup()
      .counter("myCustomCounter", new CustomCounter());
  }
}

{% endhighlight %}

#### Gauge

`Gauge`按需提供任意类型的值，要使用`Gauge`，你必须首先创建一个类并实现`org.apache.flink.metrics.Gauge`接口。
这里对返回值的类型没有限制。
可以调用`MetricGroup`的`gauge(String name,Gauge gauge)`方法来注册一个gauge。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

public class MyMapper extends RichMapFunction<String, Integer> {
  private int valueToExpose;

  @Override
  public void open(Configuration config) {
    getRuntimeContext()
      .getMetricGroup()
      .gauge("MyGauge", new Gauge<Integer>() {
        @Override
        public Integer getValue() {
          return valueToExpose;
        }
      });
  }
}

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

public class MyMapper extends RichMapFunction[String,Int] {
  val valueToExpose = 5

  override def open(parameters: Configuration): Unit = {
    getRuntimeContext()
      .getMetricGroup()
      .gauge("MyGauge", ScalaGauge[Int]( () => valueToExpose ) )
  }
  ...
}

{% endhighlight %}
</div>

</div>

请注意reporters会将暴露的对象转化成`String`型，这意味着需要去实现一个有意义的`toString（）`方法

#### Histogram
`Histogram`用于度量长值分布情况，
你可以通过调用`MetricGroup`的`histogram(String name, Histogram histogram)`方法来注册一个Histogram

{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Integer> {
  private Histogram histogram;

  @Override
  public void open(Configuration config) {
    this.histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new MyHistogram());
  }

  @public Integer map(Long value) throws Exception {
    this.histogram.update(value);
  }
}
{% endhighlight %}

 Flink没有提供一个默认的`Histogram`实现。但是提供了一个{% gh_link flink-metrics/flink-metrics-dropwizard/src/main/java/org/apache/flink/dropwizard/metrics/DropwizardHistogramWrapper.java "Wrapper" %}来允许使用 Codahale/DropWizard 直方图。
 如需使用此包装器，请在您的`pom.xml`中添加以下依赖：
{% highlight xml %}
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-dropwizard</artifactId>
      <version>{{site.version}}</version>
</dependency>
{% endhighlight %}
你可以注册一个Codahale/DropWizard 直方图类似于：

{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Integer> {
  private Histogram histogram;

  @Override
  public void open(Configuration config) {
    com.codahale.metrics.Histogram histogram =
      new com.codahale.metrics.Histogram(new SlidingWindowReservoir(500));

    this.histogram = getRuntimeContext()
      .getMetricGroup()
      .histogram("myHistogram", new DropwizardHistogramWrapper(histogram));
  }
}
{% endhighlight %}

#### Meter

`Meter`用于度量平均吞吐量，使用`markEvent（）`方法可以注册一个发生的事件。同时发生的多个事件可以使用`markEvent(long n)`方法来进行注册。
通过调用`MetricGroup`的`meter(String name, Meter meter)`方法来注册一个meter

{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Integer> {
  private Meter meter;

  @Override
  public void open(Configuration config) {
    this.meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new MyMeter());
  }

  @public Integer map(Long value) throws Exception {
    this.meter.markEvent();
  }
}
{% endhighlight %}

Flink提供了一个 {% gh_link flink-metrics/flink-metrics-dropwizard/src/main/java/org/apache/flink/dropwizard/metrics/DropwizardMeterWrapper.java "Wrapper" %}来允许使用 Codahale/DropWizard meters. 
要使用此包装器，请在您的`pom.xml`中添加以下依赖：

{% highlight xml %}
<dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-metrics-dropwizard</artifactId>
      <version>{{site.version}}</version>
</dependency>
{% endhighlight %}

您可以注册Codahale / DropWizard meter类似于这样：

{% highlight java %}
public class MyMapper extends RichMapFunction<Long, Integer> {
  private Meter meter;

  @Override
  public void open(Configuration config) {
    com.codahale.metrics.Meter meter = new com.codahale.metrics.Meter();

    this.meter = getRuntimeContext()
      .getMetricGroup()
      .meter("myMeter", new DropwizardMeterWrapper(meter));
  }
}
{% endhighlight %}

## Scope

每个指标被分配一个标识符，根据该标识符，它将基于3个组件进行报告：注册指标时用户提供的名称，可选的用户自定义域和系统提供的域。例如，如果`A.B`是系统域，`C.D`是用户域，`E`是名称，那么指标的标识符将是`A.B.C.D.E`.
你可以配置标识符的分隔符（默认:`.`）,通过设置`conf/flink-conf.yam`里面的`metrics.scope.delimiter`参数

### User Scope

你可以通过调用`MetricGroup#addGroup(String name)`和`MetricGroup#addGroup(int name)`来定义一个用户域

{% highlight java %}

counter = getRuntimeContext()
  .getMetricGroup()
  .addGroup("MyMetrics")
  .counter("myCounter");

{% endhighlight %}

### System Scope

The system scope contains context information about the metric, for example in which task it was registered or what job that task belongs to.

Which context information should be included can be configured by setting the following keys in `conf/flink-conf.yaml`.
Each of these keys expect a format string that may contain constants (e.g. "taskmanager") and variables (e.g. "&lt;task_id&gt;") which will be replaced at runtime.

系统域包含关于这个指标的上下文信息，例如其注册的任务或该任务属于哪个作业.
可以通过在`conf/flink-conf.yaml`中设置以下关键字来配置它的上下文信息。
这些关键字的每一个都期望可以包含常量的格式字符串（例如:“taskmanager”）和将在运行时被替换的变量（例如:"&lt;task_id&gt;"）

- `metrics.scope.jm`
  - 默认: &lt;host&gt;.jobmanager
  - 适用于属于一个job manager的所有指标.
- `metrics.scope.jm.job`
  - 默认: &lt;host&gt;.jobmanager.&lt;job_name&gt;
  - 适用于属于一个job manager和job的所有指标.
- `metrics.scope.tm`
  - 默认: &lt;host&gt;.taskmanager.&lt;tm_id&gt;
  - 适用于属于一个task manager的所有指标.
- `metrics.scope.tm.job`
  - 默认: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;
  - 适用于属于一个task manager或者job的所有指标.
- `metrics.scope.task`
  - 默认: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;task_name&gt;.&lt;subtask_index&gt;
   - 适用于属于一个task的所有指标.
- `metrics.scope.operator`
  - 默认: &lt;host&gt;.taskmanager.&lt;tm_id&gt;.&lt;job_name&gt;.&lt;operator_name&gt;.&lt;subtask_index&gt;
  - 适用于属于一个operator的所有指标.

这里对变量的数量和顺序没有限制，变量区分大小写.

运算指标的默认域将导致类似的标识符：

`localhost.taskmanager.1234.MyJob.MyOperator.0.MyMetric`

如果你想包含任务名称，但省略task manager信息，你可以指定以下格式：

`metrics.scope.operator: <host>.<job_name>.<task_name>.<operator_name>.<subtask_index>`

这可以创建标识符`localhost.MyJob.MySource_->_MyOperator.MyOperator.0.MyMetric`.

注意对于此格式字符串，如果同一作业同时运行多次，则可能会发生标识符冲突，导致指标标准数据不一致。
因此，建议使用格式字符串，通过包括ID（例如 &lt;job_id&gt;）
或通过为作业和操作符分配唯一的名称来提供一定程度的唯一性.

### List of all Variables

- JobManager: &lt;host&gt;
- TaskManager: &lt;host&gt;, &lt;tm_id&gt;
- Job: &lt;job_id&gt;, &lt;job_name&gt;
- Task: &lt;task_id&gt;, &lt;task_name&gt;, &lt;task_attempt_id&gt;, &lt;task_attempt_num&gt;, &lt;subtask_index&gt;
- Operator: &lt;operator_name&gt;, &lt;subtask_index&gt;

## Reporter

指标能够暴露给一个外部系统，通过在`conf/flink-conf.yaml`中配置一个或者一些reporters.
这些reporters将在每个job和task manager启动时被实例化.

- `metrics.reporters`: reporters的名称列表.
- `metrics.reporter.<name>.<config>`: 给定reporter名称`<name>`的通用设置.
- `metrics.reporter.<name>.class`: 给定reporter名称`<name>`的reporter类 .
- `metrics.reporter.<name>.interval`: 给定reporter名称`<name>`的reporter间隔.
- `metrics.reporter.<name>.scope.delimiter`: 给定reporter名称`<name>`所使用的分割分标识(默认值用：`metrics.scope.delimiter`)


所有的reporters必须至少具备`class`属性，有些允许指定一个reporting的`interval`，以下，我们将列举更多针对每个reporter的设置.

举例说明指定多个reporters的配置

```
metrics.reporters: my_jmx_reporter,my_other_reporter

metrics.reporter.my_jmx_reporter.class: org.apache.flink.metrics.jmx.JMXReporter
metrics.reporter.my_jmx_reporter.port: 9020-9040

metrics.reporter.my_other_reporter.class: org.apache.flink.metrics.graphite.GraphiteReporter
metrics.reporter.my_other_reporter.host: 192.168.1.1
metrics.reporter.my_other_reporter.port: 10000

```

**重要提示：**当Flink启动的时候，通过放入到/lib目录下包含reporter的jar文件必须可访问.
你可以通过实现`org.apache.flink.metrics.reporter.MetricReporter`接口来定义你自己的`Reporter`， 如果这个Reporter必须定期发送报告，那你也必须同时实现`Scheduled`接口.

下面的章节列举了支持的reporters.

### JMX (org.apache.flink.metrics.jmx.JMXReporter)

You don't have to include an additional dependency since the JMX reporter is available by default
but not activated.

参数:

- `port` - (可选) JMX侦听连接的端口，也可以是端口范围。当指定范围时，相关job或者task manager 日志将显示实际端口。如果设置了此设置，Flink将为给定的端口/范围启动一个额外的JMX连接器。默认本地JMX接口始终可以使用指标
你不必包含其他依赖关系，因为JMX reporter默认可用，但是没有被激活

示例配置：

{% highlight yaml %}

metrics.reporters: jmx
metrics.reporter.jmx.class: org.apache.flink.metrics.jmx.JMXReporter
metrics.reporter.jmx.port: 8789

{% endhighlight %}

Metrics exposed through JMX are identified by a domain and a list of key-properties, which together form the object name.

The domain always begins with `org.apache.flink` followed by a generalized metric identifier. In contrast to the usual
identifier it is not affected by scope-formats, does not contain any variables and is constant across jobs.
An example for such a domain would be `org.apache.flink.job.task.numBytesOut`.

The key-property list contains the values for all variables, regardless of configured scope formats, that are associated
with a given metric.
An example for such a list would be `host=localhost,job_name=MyJob,task_name=MyTask`.

The domain thus identifies a metric class, while the key-property list identifies one (or multiple) instances of that metric.

### Ganglia (org.apache.flink.metrics.ganglia.GangliaReporter)

为了使用这个reporter，你必须将`/opt/flink-metrics-ganglia-{{site.version}}.jar`拷贝到Flink的`/lib`文件夹中

参数:

- `host` - 在`gmond.conf`中的`udp_recv_channel.bind`下配置的gmond主机地址
- `port` - 在`gmond.conf`中的`udp_recv_channel.port`下配置的gmond端口
- `tmax` - 旧指标能够保留软性限制的最长时间
- `dmax` - 旧指标能够保留硬性限制的最长时间
- `ttl` - 传输UDP包的生存时间
- `addressingMode` - UDP使用的寻址模式(UNICAST/MULTICAST)  

示例配置:

{% highlight yaml %}

metrics.reporters: gang
metrics.reporter.gang.class: org.apache.flink.metrics.ganglia.GangliaReporter
metrics.reporter.gang.host: localhost
metrics.reporter.gang.port: 8649
metrics.reporter.gang.tmax: 60
metrics.reporter.gang.dmax: 0
metrics.reporter.gang.ttl: 1
metrics.reporter.gang.addressingMode: MULTICAST

{% endhighlight %}

### Graphite (org.apache.flink.metrics.graphite.GraphiteReporter)

为了使用这个reporter，你必须将`/opt/flink-metrics-graphite-{{site.version}}.jar`拷贝到Flink的`/lib`文件夹中

参数:

- `host` - Graphite 服务器地址
- `port` - Graphite 服务器端口
- `protocol` - 使用的协议 (TCP/UDP)

示例配置：

{% highlight yaml %}

metrics.reporters: grph
metrics.reporter.grph.class: org.apache.flink.metrics.graphite.GraphiteReporter
metrics.reporter.grph.host: localhost
metrics.reporter.grph.port: 2003
metrics.reporter.grph.protocol: TCP

{% endhighlight %}

### StatsD (org.apache.flink.metrics.statsd.StatsDReporter)

为了使用reporter，你必须将`/opt/flink-metrics-statsd-{{site.version}}.jar` 拷贝到Flink的`/lib`文件夹中

参数:

- `host` - the StatsD 服务器地址
- `port` - the StatsD 服务器端口

示例配置

{% highlight yaml %}

metrics.reporters: stsd
metrics.reporter.stsd.class: org.apache.flink.metrics.statsd.StatsDReporter
metrics.reporter.stsd.host: localhost
metrics.reporter.stsd.port: 8125

{% endhighlight %}

## System metrics

默认情况下，Flink收集了几个能够深入了解当前状态的指标，本章节是所有这些指标的一个参考
以下表格通常有4列：

* "Scope"列表述了用于生成系统域的域格式，例如，如果单元格包含“Operator”，则使用“metric.scope.operator”的作用域格式，如果单元格包含以斜杠分割的多个值，则会根据不同的实体报告多个指标，例如job-和askmanagers。

* "Infix"（可选） 列描述了哪些中缀附加到系统域中.

* "Metrics" 列中列出了给定域和中缀的所有注册指标的名称.

* "Description" 列提供有关给定指标度量的相关信息.

请注意，中缀/指标名称列中的所有点仍然遵循“metrics.delimiter”设置，因此，为了推断指标标识符：

1. 根据域列选择域格式
2. 添加这个值在中缀列，如果存在，则表示“metrics.delimiter”设置
3. 添加指标名称

#### CPU:
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>
      <th class="text-left" style="width: 23%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>Job-/TaskManager</strong></th>
      <td rowspan="2">Status.JVM.CPU</td>
      <td>Load</td>
      <td>JVM最近CPU使用情况.</td>
    </tr>
    <tr>
      <td>Time</td>
      <td>JVM使用的CPU时间.</td>
    </tr>
  </tbody>
</table>

#### Memory:
<table class="table table-bordered">                               
  <thead>                                                          
    <tr>                                                           
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>          
      <th class="text-left" style="width: 23%">Metrics</th>                           
      <th class="text-left" style="width: 32%">Description</th>                       
    </tr>                                                          
  </thead>                                                         
  <tbody>                                                          
    <tr>                                                           
      <th rowspan="12"><strong>Job-/TaskManager</strong></th>
      <td rowspan="12">Status.JVM.Memory</td>
      <td>Memory.Heap.Used</td>
      <td>T当前使用的堆内存大小.</td>
    </tr>
    <tr>
      <td>Heap.Committed</td>
      <td>保证JVM可用的堆内存大小.</td>
    </tr>
    <tr>
      <td>Heap.Max</td>
      <td>可用于内存管理的堆内存最大值.</td>
    </tr>
    <tr>
      <td>NonHeap.Used</td>
      <td>当前使用的非堆内存大小.</td>
    </tr>
    <tr>
      <td>NonHeap.Committed</td>
      <td>保证JVM可用的非堆内存大小.</td>
    </tr>
    <tr>
      <td>NonHeap.Max</td>
      <td>可用于内存管理的非堆内存最大值.</td>
    </tr>
    <tr>
      <td>Direct.Count</td>
      <td>直接缓冲池中的缓冲区数量.</td>
    </tr>
    <tr>
      <td>Direct.MemoryUsed</td>
      <td>JVM中用于直接缓冲池的内存大小.</td>
    </tr>
    <tr>
      <td>Direct.TotalCapacity</td>
      <td>直接缓冲池中所有缓冲区的总容量.</td>
    </tr>
    <tr>
      <td>Mapped.Count</td>
      <td>映射缓冲池中缓冲区的数量.</td>
    </tr>
    <tr>
      <td>Mapped.MemoryUsed</td>
      <td>JVM中用于映射缓冲池的内存大小.</td>
    </tr>
    <tr>
      <td>Mapped.TotalCapacity</td>
      <td>映射缓冲池中缓冲区的数量.</td>
    </tr>                                                         
  </tbody>                                                         
</table>

#### Threads:
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>
      <th class="text-left" style="width: 23%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="1"><strong>Job-/TaskManager</strong></th>
      <td rowspan="1">Status.JVM.ClassLoader</td>
      <td>Threads.Count</td>
      <td>存活线程总数.</td>
    </tr>
  </tbody>
</table>

#### GarbageCollection:
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>
      <th class="text-left" style="width: 23%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>Job-/TaskManager</strong></th>
      <td rowspan="2">Status.JVM.GarbageCollector</td>
      <td>&lt;GarbageCollector&gt;.Count</td>
      <td>已发生的回收总数.</td>
    </tr>
    <tr>
      <td>&lt;GarbageCollector&gt;.Time</td>
      <td>执行垃圾回收花费的总时间.</td>
    </tr>
  </tbody>
</table>

#### ClassLoader:
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>
      <th class="text-left" style="width: 23%">Metrics</th>
      <th class="text-left" style="width: 32%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>Job-/TaskManager</strong></th>
      <td rowspan="2">Status.JVM.ClassLoader</td>
      <td>ClassesLoaded</td>
      <td>自JVM启动以来加载类的总数.</td>
    </tr>
    <tr>
      <td>ClassesUnloaded</td>
      <td>自JVM启动以来卸载类的总数.</td>
    </tr>
  </tbody>
</table>

#### Network:
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 25%">Infix</th>
      <th class="text-left" style="width: 25%">Metrics</th>
      <th class="text-left" style="width: 30%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2"><strong>TaskManager</strong></th>
      <td rowspan="2">Status.Network</td>
      <td>AvailableMemorySegments</td>
      <td>未使用的内存段数.</td>
    </tr>
    <tr>
      <td>TotalMemorySegments</td>
      <td>已分配的内存段数.</td>
    </tr>
    <tr>
      <th rowspan="8">Task</th>
      <td rowspan="4">buffers</td>
      <td>inputQueueLength</td>
      <td>队列输入缓冲区的数量.</td>
    </tr>
    <tr>
      <td>outputQueueLength</td>
      <td>队列输出缓冲区的数量.</td>
    </tr>
    <tr>
      <td>inPoolUsage</td>
      <td>输入缓冲区使用情况评估.</td>
    </tr>
    <tr>
      <td>outPoolUsage</td>
      <td>输出缓冲区使用情况评估.</td>
    </tr>
    <tr>
      <td rowspan="4">Network.&lt;Input|Output&gt;.&lt;gate&gt;<br />
        <strong>(only available if <tt>taskmanager.net.detailed-metrics</tt> config option is set)</strong></td>
      <td>totalQueueLen</td>
      <td>所有输入/输出通道中队列缓冲区的总数.</td>
    </tr>
    <tr>
      <td>minQueueLen</td>
      <td>所有输入/输出通道中队列缓冲区的最小数目.</td>
    </tr>
    <tr>
      <td>maxQueueLen</td>
      <td>所有输入/输出通道中队列缓冲区的最大数目.</td>
    </tr>
    <tr>
      <td>avgQueueLen</td>
      <td>所有输入/输出通道中队列缓冲区的平均数目.</td>
    </tr>
  </tbody>
</table>

#### Cluster:
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 30%">Metrics</th>
      <th class="text-left" style="width: 50%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="4"><strong>JobManager</strong></th>
      <td>numRegisteredTaskManagers</td>
      <td>已注册taskmanagers的数量.</td>
    </tr>
    <tr>
      <td>numRunningJobs</td>
      <td>正在运行jobs的数量.</td>
    </tr>
    <tr>
      <td>taskSlotsAvailable</td>
      <td>可用task slots的数量</td>
    </tr>
    <tr>
      <td>taskSlotsTotal</td>
      <td>task slots总数.</td>
    </tr>
  </tbody>
</table>

#### Checkpointing:
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 30%">Metrics</th>
      <th class="text-left" style="width: 50%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="3"><strong>Job (only available on JobManager)</strong></th>
      <td>lastCheckpointDuration</td>
      <td>T完成上一次检测点所花费的时间.</td>
    </tr>
    <tr>
      <td>lastCheckpointSize</td>
      <td>上一次检测点的总大小.</td>
    </tr>
    <tr>
      <td>lastCheckpointExternalPath</td>
      <td>上一个检测点存储的路径.</td>
    </tr>
    <tr>
      <th rowspan="1">Task</th>
      <td>checkpointAlignmentTime</td>
      <td>最后一个障碍对齐所需的纳秒时间，或者当前对齐已经花费了多长时间.</td>
    </tr>
  </tbody>
</table>

#### IO:
<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Scope</th>
      <th class="text-left" style="width: 30%">Metrics</th>
      <th class="text-left" style="width: 50%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="7"><strong>Task</strong></th>
      <td>currentLowWatermark</td>
      <td>该任务已经获得的最低水位.</td>
    </tr>
    <tr>
      <td>numBytesInLocal</td>
      <td>该任务从本地源读取的字节总数.</td>
    </tr>
    <tr>
      <td>numBytesInLocalPerSecond</td>
      <td>该任务从本地源每秒读取的字节数.</td>
    </tr>
    <tr>
      <td>numBytesInRemote</td>
      <td>该任务从远端读取的字节总数.</td>
    </tr>
    <tr>
      <td>numBytesInRemotePerSecond</td>
      <td>该任务从远端每秒读取的字节数.</td>
    </tr>
    <tr>
      <td>numBytesOut</td>
      <td>该任务已发出的字节总数.</td>
    </tr>
    <tr>
      <td>numBytesOutPerSecond</td>
      <td>该任务每秒发出的字节数.</td>
    </tr>
    <tr>
      <th rowspan="4"><strong>Task/Operator</strong></th>
      <td>numRecordsIn</td>
      <td>该任务/操作已收到的条目总数.</td>
    </tr>
    <tr>
      <td>numRecordsInPerSecond</td>
      <td>该任务/操作每秒收到的条目数.</td>
    </tr>
    <tr>
      <td>numRecordsOut</td>
      <td>该操作/任务已发出的条目总数.</td>
    </tr>
    <tr>
      <td>numRecordsOutPerSecond</td>
      <td>该操作/任务每秒发出的条目数.</td>
    </tr>
    <tr>
      <th rowspan="2"><strong>Operator</strong></th>
      <td>latency</td>
      <td>所有输入源的延迟分布.</td>
    </tr>
    <tr>
      <td>numSplitsProcessed</td>
      <td>数据源已经处理的输入分片总数（如果操作是一个数据源).</td>
    </tr>
  </tbody>
</table>


### Latency tracking

Flink允许去跟踪条目在整个系统中运行的延迟，为了开启延迟跟踪，`latencyTrackingInterval `(毫秒)必须在`ExecutionConfig`中设置为一个正值.
在`latencyTrackingInterval`，源端将周期性的发送一个特殊条目，叫做`LatencyMarker`，这个标记包含一个从源端发出记录时的时间戳。延迟标记不能超过常规的用户条目，因此如果条目在一个操作的前面排队，将会通过这个标记添加延迟跟踪.

请注意延迟标记是不记录用户条目在操作中所花费的时间，而是绕过它们。特别是这个标记是不用于记录在窗口缓冲区中的时间条目。只有当操作不能够接受新的条目，它们才会排队。用这个标记测量的延迟将会反映出这一点.

所有中间操作通过保留每个源的最后`n`个延迟的列表，来计算一个延迟的分布。落地操作保留每个源的列表，然后每个并行源实例允许检测由单个机器所引起的延迟问题.

目前，Flink认为集群中所有机器的时钟是同步的。我们建议建立一个自动时钟同步服务（类似于NTP），以避免虚假的延迟结果.

### Dashboard integration

为每个任务或者操作所收集到的指标也可以在仪表盘上进行可视化。在一个作业的主页面，选择`Metrics`选项卡，在顶部图选择一个任务后，可以使用`Add Metrics`下拉菜单选择要展示的指标值
  * 任务指标被列为 `<subtask_index>.<metric_name>`.
  * 操作指标被列为 `<subtask_index>.<operator_name>.<metric_name>`.
每个指标被可视化为一个单独的图形，用x轴表示时间和y轴表示测量值。
所有的图表每10秒自动更新一次，并在导航到另一页时继续执行.

这里对可视化指标的数量没有限制；但是只有数值型指标可以可视化。

{% top %}
