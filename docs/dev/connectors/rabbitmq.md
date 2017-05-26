---
title: "RabbitMQ Connector"
nav-title: RabbitMQ
nav-parent_id: connectors
nav-pos: 6
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

* This will be replaced by the TOC
{:toc}

# RabbitMQ连接器许可证

Flink下的rabbitMQ连接器位于一个maven依赖" RabbitMQ AMQP Java Clien"上，由[Mozilla Public License v1.1 (MPL 1.1)](https://www.mozilla.org/en-US/MPL/1.1/) 许可。

Flink本身不重写" RabbitMQ AMQP Java Clien"中的源码，也不对其进行打包成二进制文件。
用户基于flink的rabbitMQ连接器（即RabbitMQ AMQP Java Clien）创建和发布拓展开的工作，可能会受到Mozilla Public License v1.1 (MPL 1.1)说明的一些限制。


# RabbitMQ连接器

该连接器访问流数据来源[RabbitMQ](http://www.rabbitmq.com/)。为使用连接器，请添加如下依赖在你的项目中，

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-rabbitmq_2.10</artifactId>
  <version>1.2.0</version>
</dependency>
{% endhighlight %}

译者注：上述方法是由maven构建项目时使用，当使用sbt构建项目时，需要在build.sbt中的正确子项目中添加如下：

{% highlight xml %}
("org.apache.flink" %% "flink-connector-rabbitmq" % flinkVersion).
              exclude("org.apache.flink","flink-shaded-hadoop1_2.10"),
如果使用scala2.10，则不需要exclude.
{% endhighlight %}

注意的是，流连接器当前都不是二元分布。更多集群执行请看[这里]({{site.baseurl}}/dev/linking.html)。

{% top %}

### 安装RabbitMQ

阅读[Rabbitmq下载页](http://www.rabbitmq.com/download.html)的介绍。安装后，服务器会自动启动，应用程序连接rabbitmq将会启动。

{% top %}

### Rabbitmq数据源

连接器提供一个类RMQSource，以消费源自于rabbitmq队列里的消息。消费RabbitMQ数据源可有三个不同层级保证，取决于flink的配置如何。

1， **仅有一次**，为实现保证仅有一次消费rabbitmq数据源，如下是需要的--

   - 可检查点：检查点生效后，在检查点完成后，消息是互相确认的（因此，会把消息从rabbitmq中删除）。

   - 使用相关编号：相关编号是rabbitmq应用的特征，当提交一个消息进rabbitmq时，必须得在消息配置中设置一个相关编号。在检查点恢复是，源利用相关编号去重已经被处理过的数据，

   - 非并行的源：实现仅有一次，源必须非并行（并行度为1）。这个限制是因为rabbitmq是从一个单一队列存在多个消费者的调度消息方式。

2， **至少一次**：当检查点生效，但是没有使用相关编号或者源是并行的，源仅仅提供至少消费一次的保证。

3， **没有保证**：如果检查点未生效，源没有任何强分发的保证。这种设置下，替代flink的检查点，一旦接收和处理消息后，消息将自动确认。

如下代码是设置成仅一次消费的例子。注释内容解释哪部分设置可忽略，以得到更多灵活保证。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(...);// 仅一次或至少一次，检查点是必须的

final RMQConnectionConfig connectionConfig = new RMQConnectionConfig.Builder()
    .setHost("localhost")
    .setPort(5000)
    ...
    .build();

final DataStream<String> stream = env
    .addSource(new RMQSource<String>(
        connectionConfig,            // rabbitmq连接的配置
        "queueName",                 // rabbitmq的队列名，消费的队列名
        true,                        // 使用相关编号，至少一次时设置为false
        new SimpleStringSchema()))   // 反序列化成java的对象
    .setParallelism(1);              // 非并行是仅一次所必须的
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.enableCheckpointing(...)

val connectionConfig = new RMQConnectionConfig.Builder()
    .setHost("localhost")
    .setPort(5000)
    ...
    .build

val stream = env
    .addSource(new RMQSource[String](
        connectionConfig,            // 配置
        "queueName",                 // 队列名
        true,                        // 使用相关编号
        new SimpleStringSchema))     // 反序列化
    .setParallelism(1)               // 非序列化
{% endhighlight %}
</div>
</div>

{% top %}

### Rabbitmq接收

连接器提供类RMQSink来发送消息到rabbitmq队列里，以下代码是一个rabbitmq接收的配置例子，

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final DataStream<String> stream = ...

final RMQConnectionConfig connectionConfig = new RMQConnectionConfig.Builder()
    .setHost("localhost")
    .setPort(5000)
    ...
    .build();

stream.addSink(new RMQSink<String>(
connectionConfig,
    "queueName",
new SimpleStringSchema()));  //序列化
val stream: DataStream[String] = ...
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val connectionConfig = new RMQConnectionConfig.Builder()
    .setHost("localhost")
    .setPort(5000)
    ...
    .build

stream.addSink(new RMQSink[String](
    connectionConfig,
"queueName",
    new SimpleStringSchema))
{% endhighlight %}
</div>
</div>

更多rabbitmq可从此[了解](http://www.rabbitmq.com/)。

{% top %}


