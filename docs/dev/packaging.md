---
title: "Program Packaging and Distributed Execution"
nav-title: Program Packaging
nav-parent_id: execution
nav-pos: 20
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


就像之前描述过的那样,Flink的程序可以用`远程环境（remote environment）`的方式执行在集群上。
同时，程序也可以选择用打成JAR包（Java Archives）的方式来执行。打包程序是使用[命令行command line interface]({{ site.baseurl }}/setup/cli.html)来运行它们的前提条件。

### 打包程序（Packaging Programs）

为了支持JAR包通过命令行或者网络接口的方式执行，程序内部必须使用包含`StreamExecutionEnvironment.getExecutionEnvironment()`的
环境。当JAR包通过命令行或者网络接口提交时，这个环境将作为集群执行的环境。如果Flink程序没有通过这些接口调用，
那么环境将表现为一个本地环境来执行。

为了打包程序，您可以简单的将所有的类导出成一个JAR包。JAR包的manifest一定要指向包含有程序
*入口点（entry point）*（有公用`main`函数）的类。最简单的一种方式就是将*main-class*加入manifest（就像这样:`main-class: org.apache.flinkexample.MyProgram`）。这个*main-class* 属性中指定的主函数与Java虚拟机中通过命令行`java -jar pathToTheJarFile`
执行JAR包时找到的主函数是一个主函数。大多数 IDE都提供了在导出JAR包时自动将这个属性包含在里面的功能。

### 通过计划来打包程序（Packaging Programs through Plans）

另外，我们还支持将程序打包成*计划（Plans)*。 计划包将返回一个用来描述程序的数据流的*程序计划（Program Plan）*，
而不是在主函数中定义一个程序并且在环境中调用`execute()`执行函数。为了做到这一点呢，程序必须要实现`org.apache.flink.api.common.Program`
接口，定义`getPlan(String...)`方法。这个方法中的字符串参数是一个命令行参数。在打包程序计划的过程中，JAR包的manifest必须指向实现的`org.apache.flinkapi.common.Program`接口，而不再是主函数类。

### 总结（Summary）

总的来说，调用一个打包的程序包含以下几个步骤：

1. JAR包的manifest寻找*main-class* 或者 *program-class*属性。如果两个属性都找到了，那*program-class* 要优先于*main-class* 属性。
当JAR的manifest文件不包含任何一个属性时，命令行或者网络接口都支持将程序入口的类名称手动传入的方式。

2. 如果入口类实现了`org.apache.flinkapi.common.Program`接口，那么系统将调用`getPlan(String...)`方法来执行并获得程序计划。

3. 如果入口类没有实现`org.apache.flinkapi.common.Program`接口，那么系统将调用类中的主函数。

{% top %}
