---
title: "Hadoop Compatibility"
is_beta: true
nav-parent_id: batch
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

Flink兼容Apache Hadoop MapReduce的接口，因此可以使用面向MapReduce的代码。

你可以:

- Flink中使用Hadoop `Writable` [data types](index.html#data-types).
- 使用Hadoop `InputFormat` 作为[DataSource](index.html#data-sources).
- 使用Hadoop `OutputFormat` 作为 a [DataSink](index.html#data-sinks).
- 使用Hadoop `Mapper` 作为 [FlatMapFunction](dataset_transformations.html#flatmap).
- 使用Hadoop `Reducer` 作为 [GroupReduceFunction](dataset_transformations.html#groupreduce-on-grouped-dataset).

这篇文档展示如何在Flink中使用现存的Hadoop MapReduce代码。可以参考
[连接其他系统]({{ site.baseurl }}/dev/batch/connectors.html) 来了解如何从Hadoop支持的文件系统中读取数据。

* This will be replaced by the TOC
{:toc}

### 项目配置

支持Hadoop的input／output格式是`flink-java`和`flink-scala`的maven模块的一部分，这两部分是在编写Flink任务时经常需要用到的。 `mapred`和`mapreduce` 的api代码分别在`org.apache.flink.api.java.hadoop`和`org.apache.flink.api.scala.hadoop`以及一个额外的子package中。

对Hadoop MapReduce的支持是在`flink-hadoop-compatibility`的maven模块中。代码具体在`org.apache.flink.hadoopcompatibility`包中。

如果想要重复使用`Mappers and Reducers`， 需要在maven中添加下面依赖：

~~~xml
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-hadoop-compatibility{{ site.scala_version_suffix }}</artifactId>
	<version>{{site.version}}</version>
</dependency>
~~~

### 使用Hadoop数据类型

Flink支持所有的Hadoop `Writable` 和 `WritableComparable` 数据类型, 不用额外添加Hadoop Compatibility 依赖。 可以参考[Programming Guide](index.html#data-types)了解如何使用Hadoop数据类型（Hadoop data type）。

### 使用Hadoop输入格式

可以使用Hadoop输入格式来创建数据源，具体是调用 ExecutionEnvironment 的 readHadoopFile 或 createHadoopInput方法。 前者用于来自FileInputFormat的输入格式， 后者用于普通的输入格式。

创建的数据集包含的是一个“键-值”2元组，“值”是从Hadoop输入格式获得的数值。

下面的例子介绍如何使用Hadoop的 `TextInputFormat`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Tuple2<LongWritable, Text>> input =
    env.readHadoopFile(new TextInputFormat(), LongWritable.class, Text.class, textPath);

// Do something with the data.
[...]
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val env = ExecutionEnvironment.getExecutionEnvironment

val input: DataSet[(LongWritable, Text)] =
  env.readHadoopFile(new TextInputFormat, classOf[LongWritable], classOf[Text], textPath)

// Do something with the data.
[...]
~~~

</div>

</div>

### 使用Hadoop输出格式

Flink提供兼容Hadoop输出格式（Hadoop OutputFormat）的封装。支持任何实现`org.apache.hadoop.mapred.OutputFormat`接口或者继承`org.apache.hadoop.mapreduce.OutputFormat`的类。输出格式的封装需要的输入是“键值对”形式。他们将会交给Hadoop输出格式处理。

下面的例子介绍如何使用Hadoop的 `TextOutputFormat`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// Obtain the result we want to emit
DataSet<Tuple2<Text, IntWritable>> hadoopResult = [...]

// 创建和初始化Hadoop TextOutputFormat.
HadoopOutputFormat<Text, IntWritable> hadoopOF =
  new HadoopOutputFormat<Text, IntWritable>(
    // 设置Hadoop OutputFormat和特定的job作为初始化参数
    new TextOutputFormat<Text, IntWritable>(), job
  );
hadoopOF.getConfiguration().set("mapreduce.output.textoutputformat.separator", " ");
TextOutputFormat.setOutputPath(job, new Path(outputPath));

// 通过Hadoop TextOutputFormat发布数据
hadoopResult.output(hadoopOF);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// Obtain your result to emit.
val hadoopResult: DataSet[(Text, IntWritable)] = [...]

val hadoopOF = new HadoopOutputFormat[Text,IntWritable](
  new TextOutputFormat[Text, IntWritable],
  new JobConf)

hadoopOF.getJobConf.set("mapred.textoutputformat.separator", " ")
FileOutputFormat.setOutputPath(hadoopOF.getJobConf, new Path(resultPath))

hadoopResult.output(hadoopOF)


~~~

</div>

</div>

### 使用Hadoop Mappers和Reducers 

`Hadoop Mappers` 语法上等价于Flink的`FlatMapFunctions`，`Hadoop Reducers`语法上等价于Flink的`GroupReduceFunctions`。 Flink同样封装了`Hadoop MapReduce`的`Mapper and Reducer`接口的实现。 用户可以在Flink程序中复用Hadoop的`Mappers and Reducers`。 这时，仅仅`org.apache.hadoop.mapred`的Mapper and Reducer接口被支持。

The wrappers take a `DataSet<Tuple2<KEYIN,VALUEIN>>` as input and produce a `DataSet<Tuple2<KEYOUT,VALUEOUT>>` as output where `KEYIN` and `KEYOUT` are the keys and `VALUEIN` and `VALUEOUT` are the values of the Hadoop key-value pairs that are processed by the Hadoop functions. For Reducers, Flink offers a wrapper for a GroupReduceFunction with (`HadoopReduceCombineFunction`) and without a Combiner (`HadoopReduceFunction`). The wrappers accept an optional `JobConf` object to configure the Hadoop Mapper or Reducer.

封装函数用`DataSet<Tuple2<KEYIN,VALUEIN>>`作为输入， 产生`DataSet<Tuple2<KEYOUT,VALUEOUT>>`作为输出， 其中`KEYIN`和`KEYOUT`是“键” ，`VALUEIN` 和`VALUEOUT` 是“值”，它们是Hadoop函数处理的键值对。 对于Reducers，Flink将GroupReduceFunction封装成`HadoopReduceCombineFunction`，但没有Combiner(`HadoopReduceFunction`)。 封装函数接收可选的`JobConf`对象来配置Hadoop的Mapper or Reducer。

Flink的方法封装有

- `org.apache.flink.hadoopcompatibility.mapred.HadoopMapFunction`,
- `org.apache.flink.hadoopcompatibility.mapred.HadoopReduceFunction`, and
- `org.apache.flink.hadoopcompatibility.mapred.HadoopReduceCombineFunction`.
他们可以被用于[FlatMapFunctions](dataset_transformations.html#flatmap)或[GroupReduceFunctions](dataset_transformations.html#groupreduce-on-grouped-dataset).

下面的例子介绍如何使用Hadoop的`Mapper`和`Reducer` 。

~~~java
// Obtain data to process somehow.
DataSet<Tuple2<Text, LongWritable>> text = [...]

DataSet<Tuple2<Text, LongWritable>> result = text
  // 使用Hadoop Mapper (Tokenizer)作为Map函数
  .flatMap(new HadoopMapFunction<LongWritable, Text, Text, LongWritable>(
    new Tokenizer()
  ))
  .groupBy(0)
  // 使用Hadoop Reducer (Counter)作为Reduce函数
  .reduceGroup(new HadoopReduceCombineFunction<Text, LongWritable, Text, LongWritable>(
    new Counter(), new Counter()
  ));
~~~

**需要注意:** Reducer封装处理由Flink中的[groupBy()](dataset_transformations.html#transformations-on-grouped-dataset)定义的groups。 它并不考虑任何在JobConf定义的自定义的分区器(partitioners), 排序（sort）或分组（grouping）的比较器。

### 完整Hadoop WordCount示例

下面给出一个完整的使用Hadoop 数据类型， InputFormat/OutputFormat/Mapper/Reducer的示例。

~~~java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// 创建和初始化Hadoop TextInputFormat.
Job job = Job.getInstance();
HadoopInputFormat<LongWritable, Text> hadoopIF =
  new HadoopInputFormat<LongWritable, Text>(
    new TextInputFormat(), LongWritable.class, Text.class, job
  );
TextInputFormat.addInputPath(job, new Path(inputPath));

// 从Hadoop TextInputFormat读取数据.
DataSet<Tuple2<LongWritable, Text>> text = env.createInput(hadoopIF);

DataSet<Tuple2<Text, LongWritable>> result = text
  // 使用Hadoop Mapper (Tokenizer)作为Map函数
  .flatMap(new HadoopMapFunction<LongWritable, Text, Text, LongWritable>(
    new Tokenizer()
  ))
  .groupBy(0)
  // 使用Hadoop Reducer (Counter)作为Reduce函数
  .reduceGroup(new HadoopReduceCombineFunction<Text, LongWritable, Text, LongWritable>(
    new Counter(), new Counter()
  ));

// 创建和初始化Hadoop TextOutputFormat.
HadoopOutputFormat<Text, IntWritable> hadoopOF =
  new HadoopOutputFormat<Text, IntWritable>(
    new TextOutputFormat<Text, IntWritable>(), job
  );
hadoopOF.getConfiguration().set("mapreduce.output.textoutputformat.separator", " ");
TextOutputFormat.setOutputPath(job, new Path(outputPath));

// 使用the Hadoop TextOutputFormat输出结果.
result.output(hadoopOF);

// 执行程序
env.execute("Hadoop WordCount");
~~~
