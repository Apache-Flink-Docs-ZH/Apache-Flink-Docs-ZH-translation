---
title: "Apache NiFi Connector"
nav-title: NiFi
nav-parent_id: connectors
nav-pos: 7
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

此连接器提供可以读取和写入[Apache NiFi](https://nifi.apache.org/)的源（Source）和槽（Sink）. 要使用此连接器，请将以下依赖项添加到您的项目中：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-nifi{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

请注意，流连接器当前不是二进制分发的一部分。有关如何将程序与程序库打包以进行集群执行的信息，请参阅
[此处]({{site.baseurl}}/dev/linking.html)。

#### 安装 Apache NiFi

有关设置Apache NiFi集群的说明可以在[这里](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html#how-to-install-and-start-nifi)找到。

#### Apache NiFi 源

该连接器提供了从Apache NiFi到Apache Flink读取数据的源。

`NiFiSource(…)`类提供2个构建器（constructors），用于从NiFi读取数据。

- `NiFiSource(SiteToSiteConfig config)` - 为指定客户端的SiteToSiteConfig构建一个`NiFiSource(…)`，默认等待时间为1000 ms。

- `NiFiSource(SiteToSiteConfig config, long waitTimeMs)` - 为指定客户端的SiteToSiteConfig和指定的等待时间（以毫秒为单位）构建一个`NiFiSource(…)`。

示例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment streamExecEnv = StreamExecutionEnvironment.getExecutionEnvironment();

SiteToSiteClientConfig clientConfig = new SiteToSiteClient.Builder()
        .url("http://localhost:8080/nifi")
        .portName("Data for Flink")
        .requestBatchCount(5)
        .buildConfig();

SourceFunction<NiFiDataPacket> nifiSource = new NiFiSource(clientConfig);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val streamExecEnv = StreamExecutionEnvironment.getExecutionEnvironment()

val clientConfig: SiteToSiteClientConfig = new SiteToSiteClient.Builder()
       .url("http://localhost:8080/nifi")
       .portName("Data for Flink")
       .requestBatchCount(5)
       .buildConfig()

val nifiSource = new NiFiSource(clientConfig)       
{% endhighlight %}       
</div>
</div>

这里的数据从Apache NiFi输出端口读取，该端口称为“Data for Flink”，这是Apache NiFi站点到站点协议配置的一部分。

#### Apache NiFi 槽（Sink）

连接器提供了一个槽（Sink），用于将数据从Apache Flink写入Apache NiFi。

`NiFiSink(…)` 类提供了一个实例化`NiFiSink`的构造函数。

- `NiFiSink(SiteToSiteClientConfig, NiFiDataPacketBuilder<T>)`为指定客户端的`SiteToSiteConfig`和`NiFiDataPacketBuilder`构造了一个`NiFiSink(…)`，它将数据从Flink转换为`NiFiDataPacket`，将由NiFi进行获取。

示例：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment streamExecEnv = StreamExecutionEnvironment.getExecutionEnvironment();

SiteToSiteClientConfig clientConfig = new SiteToSiteClient.Builder()
        .url("http://localhost:8080/nifi")
        .portName("Data from Flink")
        .requestBatchCount(5)
        .buildConfig();

SinkFunction<NiFiDataPacket> nifiSink = new NiFiSink<>(clientConfig, new NiFiDataPacketBuilder<T>() {...});

streamExecEnv.addSink(nifiSink);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val streamExecEnv = StreamExecutionEnvironment.getExecutionEnvironment()

val clientConfig: SiteToSiteClientConfig = new SiteToSiteClient.Builder()
       .url("http://localhost:8080/nifi")
       .portName("Data from Flink")
       .requestBatchCount(5)
       .buildConfig()

val nifiSink: NiFiSink[NiFiDataPacket] = new NiFiSink[NiFiDataPacket](clientConfig, new NiFiDataPacketBuilder<T>() {...})

streamExecEnv.addSink(nifiSink)
{% endhighlight %}       
</div>
</div>      

有关 [Apache NiFi](https://nifi.apache.org)站点到站点协议的更多信息，请点击 [此处](https://nifi.apache.org/docs/nifi-docs/html/user-guide.html#site-to-site)。
