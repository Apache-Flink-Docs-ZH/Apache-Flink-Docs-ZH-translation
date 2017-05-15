---
title: "并行执行"
nav-parent_id: execution
nav-pos: 30
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


本节描述了如何在Flink中配置程序的并行执行。一个Flink程序由多个任务组成(变换/操作符, 数据源和 sinks)。
一个任务被切分为多个并行的实例来执行，而每一个并行的实例处理任务输入数据的一个子集。
一个任务的并行实例数目就被称为该任务的并行度。


如果你想使用[savepoints]({{ site.baseurl }}/setup/savepoints.html)，你应该同时考虑设置最大的并行度。当从一个savepoint点
恢复时，你可以改变特定算子或着整个程序的并行度，同时这个设置指定了并行度的上限。为了不降低性能，这个设置是必须的；因为Flink 内部将状态划分为key-groups，但我们不能无限制的增加key-groups。

* toc
{:toc}

## 设置并行度

一个任务的并行度可以从多个层次指定：

## 算子层次


一个算子、数据源和sink的并行度可以通过调用 `setParallelism()`方法来指定。就像这样：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = text
    .flatMap(new LineSplitter())
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5);

wordCounts.print();

env.execute("Word Count Example");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1).setParallelism(5)
wordCounts.print()

env.execute("Word Count Example")
{% endhighlight %}
</div>
</div>


### 执行环境层次

就像[这里]({{ site.baseurl }}/dev/api_concepts.html#anatomy-of-a-flink-program)描述的，Flink程序运行在执行环境的
上下文中。执行环境为所有执行的算子、数据源、data sink定义了一个默认的并行度。执行环境的并行度可以通过显式配置
算子的并行度而被重写。

执行环境的默认并行度可以通过调用`setParallelism()`方法指定。为了以并行度3来执行所有的算子、数据源和data sink，
可以通过如下的方式设置执行环境的并行度：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(3);

DataStream<String> text = [...]
DataStream<Tuple2<String, Integer>> wordCounts = [...]
wordCounts.print();

env.execute("Word Count Example");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(3)

val text = [...]
val wordCounts = text
    .flatMap{ _.split(" ") map { (_, 1) } }
    .keyBy(0)
    .timeWindow(Time.seconds(5))
    .sum(1)
wordCounts.print()

env.execute("Word Count Example")
{% endhighlight %}
</div>
</div>


### 客户端层次

并行度可以在客户端将job提交到Flink时设定。客户端可以是Java或Scala程序，典型的例子是Flink的命令行接口(CLI).


对于CLI客户端，可以通过-p参数指定并行度。例如：

    ./bin/flink run -p 10 ../examples/*WordCount-java*.jar

在Java/Scala程序中，并行度可以这样设置：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

try {
    PackagedProgram program = new PackagedProgram(file, args);
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123");
    Configuration config = new Configuration();

    Client client = new Client(jobManagerAddress, config, program.getUserCodeClassLoader());

    // set the parallelism to 10 here
    client.run(program, 10, true);

} catch (ProgramInvocationException e) {
    e.printStackTrace();
}

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
try {
    PackagedProgram program = new PackagedProgram(file, args)
    InetSocketAddress jobManagerAddress = RemoteExecutor.getInetFromHostport("localhost:6123")
    Configuration config = new Configuration()

    Client client = new Client(jobManagerAddress, new Configuration(), program.getUserCodeClassLoader())

    // set the parallelism to 10 here
    client.run(program, 10, true)

} catch {
    case e: Exception => e.printStackTrace
}
{% endhighlight %}
</div>
</div>


### 系统层次

在系统级可以通过设置./conf/flink-conf.yaml`文件中的`parallelism.default` 属性来指定所有执行环境的默认并行度。
可以通过查看[配置]({{ site.baseurl }}/setup/config.html)文档以获取更详细的细节。

## 设置最大并行度

最大并行度可以在设置并行度的地方设定(除了客户端和系统层次)。不同于调用`setParallelism()`方法，
你可以通过调用`setMaxParallelism()`方法来设定最大并行度。

默认的最大并行度大概等于‘算子的并行度+算子的并行度/2’，其下限为127而上限为32768。

<span class="label label-danger">注意</span> 设置最大并行度到一个非常大的值将会降低性能因为一些状态的后台需要维持内部的数据结构，而这些数据结构将会随着key-groups的数目而扩张（key-groups 是rescalable状态的内部实现机制）。

{% top %}
