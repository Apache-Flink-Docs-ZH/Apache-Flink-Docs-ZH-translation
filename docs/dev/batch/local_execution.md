---
title:  "Local Execution"
nav-parent_id: batch
nav-pos: 8
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

Flink可以在单个机器上运行，甚至在单个Java虚拟机中。这允许用户在本地测试和调试Flink程序。本节概述了本地执行机制。

本地环境和执行器允许你在本地Java虚拟机中运行Flink程序，也可以在任何JVM作为现有程序的一部分来运行Flink程序。大多数示例可以通过简单地点击IDE的“运行”按钮在本地启动。

Flink支持两种不同的本地执行支持。 `本地执行环境`启动完整的Flink运行时，包括JobManager和TaskManager。这些包括内存管理和在群集模式下执行的所有内部算法。


`集合环境`在Java集合上执行Flink程序。此模式将不会启动完整的Flink运行时，因此这种执行方式开销非常低并且轻量级。例如，将通过将“map（）”函数应用于Java list中的所有元素来执行`DataSet.map（）`转换。

* TOC
{:toc}


## 调试

如果您在本地运行Flink程序，还可以像调试其他Java程序一样调试你的程序。您可以使用`System.out.println（）`来输出一些内部变量，也可以使用调试器。可以在`map（）`，`reduce（）`和所有其他方法中设置断点。请参阅Java API文档中的[调试部分]({{ site.baseurl }}/dev/batch/index.html#debugging) ，以获取Java API中测试和本地调试实用程序的指南。

## Maven 依赖

如果您正在Maven项目中开发您的程序，则必须使用以下依赖关系添加`flink-clients`模块：

~~~xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version}}</version>
</dependency>
~~~

## 本地环境

`本地环境`是Flink程序的本地执行的句柄。使用它在本地JVM中运行程序 - 独立或嵌入其他程序。

本地环境通过`ExecutionEnvironment.createLocalEnvironment（）`方法来实例化。默认情况下，它将使用像你机器CPU核数一样多的本地线程来执行，（硬件上下文）。您也可以指定所需的并行性。本地环境可以通过配置`enableLogging（）`/`disableLogging（）`来记录日志到控制台。

在大多数情况下，调用`ExecutionEnvironment.getExecutionEnvironment（）`是更好的方法。当程序在本地启动（命令行界面外）时，该方法返回一个`本地环境`，并且当程序被[命令行界面]({{ site.baseurl }}/setup/cli.html)。

~~~java
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

    DataSet<String> data = env.readTextFile("file:///path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("file:///path/to/result");

    JobExecutionResult res = env.execute();
}
~~~


在执行完成后返回的`JobExecutionResult`对象包含程序运行时和累加器结果。


`本地环境`还允许将自定义配置值传递给Flink。

~~~java
Configuration conf = new Configuration();
conf.setFloat(ConfigConstants.TASK_MANAGER_MEMORY_FRACTION_KEY, 0.5f);
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment(conf);
~~~

*注意：*本地执行环境不会启动任何Web前端来监视执行。

## 集合环境

使用`集合环境`在Java集合上的执行是执行Flink程序的低开销方法。此模式的典型用例是自动测试，调试和代码重用。

用户可以在批处理中，甚至是更具互动性的case中使用算法实现。 Flink程序的一个稍微改变的变体可以在Java应用服务器中用于处理传入的请求。

**基于集合执行的代码框架**

~~~java
public static void main(String[] args) throws Exception {
    // 初始化一个新的基于集合的执行环境
    final ExecutionEnvironment env = new CollectionEnvironment();

    DataSet<User> users = env.fromCollection( /* 从Java集合中获取元素 */);

    /* 数据及转换 ... */

    // 将生成的Tuple2元素存入一个ArrayList中。
    Collection<...> result = new ArrayList<...>();
    resultDataSet.output(new LocalCollectionOutputFormat<...>(result));

    // 开始执行。
    env.execute();

    // 对于生成的ArrayList执行一些操作 (=集合).
    for(... t : result) {
        System.err.println("Result = "+t);
    }
}
~~~

`flink-examples-batch`模块包含一个完整的例子，叫做`CollectionExecutionExample`。

请注意，基于集合的Flink程序的执行仅适用于在JVM堆中的少量数据。集合上的执行不是多线程的，只使用一个线程。
