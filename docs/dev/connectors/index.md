---
title: "Streaming Connectors"
nav-id: connectors
nav-title: Connectors
nav-parent_id: streaming
nav-pos: 30
nav-show_overview: true
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

Connectors 模块提供了与各种第三方系统连接的代码。

当前，只支持下面的系统：（请从左侧导航栏中选择相应的文档。）

 * [Apache Kafka](https://kafka.apache.org/) (sink/source)
 * [Elasticsearch](https://elastic.co/) (sink)
 * [Hadoop FileSystem](http://hadoop.apache.org) (sink)
 * [RabbitMQ](http://www.rabbitmq.com/) (sink/source)
 * [Amazon Kinesis Streams](http://aws.amazon.com/kinesis/streams/) (sink/source)
 * [Twitter Streaming API](https://dev.twitter.com/docs/streaming-apis) (source)
 * [Apache NiFi](https://nifi.apache.org) (sink/source)
 * [Apache Cassandra](https://cassandra.apache.org/) (sink)


如果需要在应用程序里面使用到其中一个 connectors，一般需要安装和启动额外的第三方组件，比如消息队列系统。有关这些系统的进一步说明可以在相应的小节中找到。