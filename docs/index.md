---
title: "Apache Flink 中文文档"
nav-pos: 0
nav-title: '<i class="fa fa-home title" aria-hidden="true"></i> Home'
nav-parent_id: root
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

本文档是针对 Apache Flink {{ site.version }} 的，本页面的编译时间: {% build_time %}。

Apache Flink 是一个开源的分布式流处理和批处理系统。Flink 的核心是在数据流上提供数据分发、通信、具备容错的分布式计算。同时，Flink 在流处理引擎上构建了批处理引擎，原生支持了迭代计算、内存管理和程序优化。

## 第一步

- **概念**: 从 Flink 的[数据流编程模型](concepts/programming-model.html)和[分布式运行环境](concepts/runtime.html)的基本概念开始。 这将有助于您充分了解其他部分的文档，包括安装以及编程指南。强烈推荐先阅读这部分的文档。

- **快速起步**: 在你的本地机器上[运行一个实例](quickstart/setup_quickstart.html) 或者 [编写一个简单的程序](quickstart/run_example_quickstart.html) 来操作 Wikipedia 的编辑日志。

- **编程指南**: 你可以在本指南里面找到一些 [基本概念](dev/api_concepts.html) 和 [DataStream API](dev/datastream_api.html) 或者 [DataSet API](dev/batch/index.html) 学习如何编写第一个 Flink 程序。

## 迁移指南

对于那些使用比较早期版本的 Apache Flink 用户，我们推荐你阅读 [API 迁移指南](dev/migration.html)。
虽然 API 中标记为 public 和 stable 的所有部分仍然被支持 (标记为 public 的 API 是向后兼容的)，我们仍然建议将应用程序迁移到较新接口。

对于计划在生产环境中升级 Flink 的用户，我们推荐你阅读[升级 Apache Flink](ops/upgrading.html)指南。
