---
title: "基础的API概念"
nav-parent_id: dev
nav-pos: 1
nav-show_overview: true
nav-id: api-concepts
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

Flink是实现分布式数据集合转换的通用程序(例如过滤、映射、更新状态、连结、分组、定义窗口、聚合等)。数据集合最先是从源（例如文件、kafka主题或者本地内存数据集合）创建。经过接收器之后得到结果，例如可以把数据写入到(分布式)文件中，或者执行标准输出（例如，命令行终端）。Flink程序支持各种运行方式，可以独立运行，也可以嵌入到其他程序中。可以在本地JVM或设备集群上执行。

根据数据源的类型，即有界或无界的数据源，你可以使用DataSet API编写批处理程序或者DataStream API编写流处理程序。本篇将会介绍这两种API共有的基本概念，但如果你想获取每种API具体的编写教程，请参阅我们的[流处理教程]({{ site.baseurl }}/dev/datastream_api.html) 和[批处理教程]({{ site.baseurl }}/dev/batch/index.html)。

**NOTE:** When showing actual examples of how the APIs can be used  we will use
`StreamingExecutionEnvironment` and the `DataStream` API. The concepts are exactly the same
in the `DataSet` API, just replace by `ExecutionEnvironment` and `DataSet`.

**注意：**当我们在实际的例子中展示各个API如何使用事，我们将会使用`StreamingExecutionEnvironment` 和`DataStream` API。这和使用`DataSet` API很相似，直接替换成`ExecutionEnvironment` 和`DataSet`即可。

* This will be replaced by the TOC
{:toc}

批量数据和 流式数据
----------------------

Flink 有特定的`DataSet` 和`DataStream` 类来表示程序中的数据。你可以把它们视为能包含重复项的不可变数据集合。在这种情况下`DataSet` 的数据是有限的， 而对于`DataStream` ，元素的数量可以试无限的。

这些集合在一些关键方面与Java集合不同，首先，它们是不可变的，这就说明一旦它们被创建了，就不再允许你新增或删除元素。同时，你不能简单的检查里面的元素。

一个集合最初是通过在Flink程序里添加一个源来创建的，通过使用API方法（如`map`,`filter` 等）把它们进行转换，从而派生出新的集合。

Flink程序剖析
--------------------------

Flink程序编码看起来就像常规的数据集合转换编码。每个程序由相同的基本部分组成：

1. 获取一个运行环境`execution environment`，
2. 加载/创建初始数据，
3. 指定这些数据的转换，
4. 指定放置计算结果的位置，
5. 触发程序执行

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

现在我们将概述其中的每一个步骤，请参阅各个章节以获取更多详细信息。请注意，Java DataSet API的所有核心类可在{% gh_link /flink-java/src/main/java/org/apache/flink/api/java "org.apache.flink.api.java" %}包中找到 ，而Java DataStream API的类可在 {% gh_link /flink-streaming-java/src/main/java/org/apache/flink/streaming/api "org.apache.flink.streaming.api" %}中找到。

`StreamExecutionEnvironment` 是所有Flink程序的基础。你可以使用以下静态方法获得一个`StreamExecutionEnvironment` ：

{% highlight java %}
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(String host, int port, String... jarFiles)
{% endhighlight %}

通常情况下，您只需要使用`getExecutionEnvironment()`方法，因为它将会根据上下文做正确的事情：如果您在IDE内执行程序或作为常规Java程序执行，它将创建一个本地环境，以在本地计算机上执行您的程序。如果您从程序创建JAR文件并通过[命令行]({{ site.baseurl }}/setup/cli.html)调用它 ，则Flink集群管理器将执行您的main方法，`getExecutionEnvironment()`将会返回在集群下执行程序的运行环境。

为了指定数据源，运行环境有多种方法可以使用各种方法来通过不同的方式读取文件：你可以逐行读取它们，以CSV文件的形式读取它们，或使用完全自定义的数据输入格式。要仅将文本文件作为一系列行读取，可以使用：

{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = env.readTextFile("file:///path/to/file");
{% endhighlight %}

这将为您提供一个DataStream，然后您可以将其转换来创建新的派生的DataStream。

您可以通过调用DataStream的转换函数来进行转换，例如，一个map转换如下所示：

{% highlight java %}
DataStream<String> input = ...;

DataStream<Integer> parsed = input.map(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});
{% endhighlight %}

这将通过将原始集合中的每个String转换为Integer来创建新的DataStream。

一旦有了包含最终结果的DataStream，就可以通过创建接收器将其写入外部系统。这些只是创建接收器的一些示例方法：

{% highlight java %}
writeAsText(String path)

print()
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

现在我们将概述其中的每一个步骤，请参阅各个章节以获取更多详细信息。请注意，Scala DataSet API的所有核心类都可以在{% gh_link /flink-scala/src/main/scala/org/apache/flink/api/scala "org.apache.flink.api.scala" %}包中找到， 而Scala DataStream API的类可以在{% gh_link /flink-streaming-scala/src/main/scala/org/apache/flink/streaming/api/scala "org.apache.flink.streaming.api.scala" %}中找到。

`StreamExecutionEnvironment` 是所有Flink程序的基础，您可以使用以下静态方法获得一个`StreamExecutionEnvironment`：

{% highlight scala %}
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(host: String, port: Int, jarFiles: String*)
{% endhighlight %}

通常情况下，您只需要使用`getExecutionEnvironment()`方法，因为它将会根据上下文做正确的事情：如果您在IDE内执行程序或作为常规Java程序执行，它将创建一个本地环境，以在本地计算机上执行您的程序。如果您从程序创建JAR文件并通过[命令行]({{ site.baseurl }}/setup/cli.html)调用它 ，则Flink集群管理器将执行您的main方法，`getExecutionEnvironment()`将会返回在集群下执行程序的运行环境。

为了指定数据源，运行环境有多种方法可以使用各种方法来通过不同的方式读取文件：你可以逐行读取它们，以CSV文件的形式读取它们，或使用完全自定义的数据输入格式。要仅将文本文件作为一系列行读取，可以使用：

{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment()

val text: DataStream[String] = env.readTextFile("file:///path/to/file")
{% endhighlight %}

这将为您提供一个DataStream，然后您可以将其转换来创建新的派生的DataStream。

您可以通过调用DataStream的转换函数来进行转换，例如，一个map转换如下所示：

{% highlight scala %}
val input: DataSet[String] = ...

val mapped = input.map { x => x.toInt }
{% endhighlight %}

这将通过将原始集合中的每个String转换为Integer来创建新的DataStream。

一旦有了包含最终结果的DataStream，就可以通过创建接收器将其写入外部系统。这些只是创建接收器的一些示例方法：

{% highlight scala %}
writeAsText(path: String)

print()
{% endhighlight %}

</div>
</div>

当您按照上面几步写好完整的程序，您现在需要调用`StreamExecutionEnvironment`的`execute()`方法来触发程序执行。根据`ExecutionEnvironment` 的类型，程序将会在您的本地设备或被提交到集群运行环境中触发执行。

该`execute()`方法返回一个`JobExecutionResult`，它包含执行时间和累加器的结果。

请参阅[流数据处理指南]({{ site.baseurl }}/dev/datastream_api.html)获取关于流数据源和接收器的信息以及更深入的关于DataStream支持的转换等信息。

请参阅[批数据处理指南]({{ site.baseurl }}/dev/batch/index.html)获取关于批数据源和接收器的信息以及更深入的关于DataSet支持的转换等信息。


{% top %}

惰性算法
---------------

所有Flink程序都会被惰性执行：当程序的main方法被执行时，数据加载和转换不会直接发生。相反，每个操作都会创建并添加到程序的计划中。当运行环境调用`execute()`方法时才会触发执行实际操作。程序是在本地执行还是在群集上执行取决于运行环境的类型。

惰性算法让您可以构建Flink作为整体计划单元执行的复杂程序。

{% top %}

指定键
---------------

一些转换（join, coGroup, keyBy, groupBy）需要定义键的集合。其他转换（Reduce, GroupReduce,
Aggregate, Windows）允许数据在应用之前被一个键分组。

如下所示一个DataSet被分组

{% highlight java %}
DataSet<...> input = // [...]
DataSet<...> reduced = input
  .groupBy(/*define key here*/)
  .reduceGroup(/*do something*/);
{% endhighlight %}

而使用DataStream时可以指定一个键

{% highlight java %}
DataStream<...> input = // [...]
DataStream<...> windowed = input
  .keyBy(/*define key here*/)
  .window(/*window specification*/);
{% endhighlight %}

Flink的数据模型不基于键值对。因此，您不需要将数据集类型物理地打包成键和值。这里keys分组的概念是“虚拟的”：它们被定义为实际数据之上的函数来进行分组操作。 

**注意：**在接下来的讨论中我们将会使用`DataStream` API和`keyBy`。如果想使用DataSet API请您直接替换为 `DataSet` 和`groupBy`即可。

### 定义元组的键
{:.no_toc}

最简单的情况是根据元组的一个或多个字段上将元组进行分组： 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0)
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[(Int, String, Long)] = // [...]
val keyed = input.keyBy(0)
{% endhighlight %}
</div>
</div>

元组根据第一个字段（Integer类型的一个字段）进行分组。 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0,1)
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataSet[(Int, String, Long)] = // [...]
val grouped = input.groupBy(0,1)
{% endhighlight %}
</div>
</div>

在这里，我们使用元组的第一个和第二个字段组成的组合键将元组进行分组

关于嵌套元组的说明：如果您有一个嵌套元组的DataStream，例如： 

{% highlight java %}
DataStream<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
{% endhighlight %}

指定`keyBy(0)`将导致系统将整个`Tuple2`用作键（以Integer和Float为键）。如果你想“导航”到嵌套的`Tuple2`里，你必须使用下面介绍的字段表达式来定义键。 

### 使用字段表达式定义键
{:.no_toc}

您可以使用基于字符串的字段表达式来引用嵌套字段并定义用于分组，排序，连接或共同组的键。 

字段表达式使我们可以很容易的选择（嵌套）复杂类型的字段，例如[元组](#tuples-and-case-classes)和[Java对象](#pojos)

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

下面的例子中，我们有一个对象`WC`，它有 "word" 和"count"两个字段，如果根据字段`word`进行分组，我们可以直接把它的名字传入`keyBy()`函数中。

{% highlight java %}
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word;
  public int count;
}
DataStream<WC> words = // [...]
DataStream<WC> wordCounts = words.keyBy("word").window(/*window specification*/);
{% endhighlight %}

**字段表达式分析：**

- 按字段名称选择POJO字段。例如，`"user"`指的是POJO类型的“用户”字段。
- 按字段名称或0偏移量字段索引选择元组字段。例如`"f0"`和`"5"`参考Java Tuple类型的第一个和第六个字段。
- 您可以选择POJO和元组中的嵌套字段。例如，`"user.zip"`指的是存储在POJO类型的“用户”中的“zip”字段。支持POJO和元组的任意嵌套和混合，例如`"f1.user.zip"`或`"user.f3.1.zip"`。
- 您可以使用`"*"`通配符表达式来选择完整类型。这也适用于不是Tuple或POJO类型的类型。 

**字段表达式示例：**

{% highlight java %}
public static class WC {
  public ComplexNestedClass complex; //nested POJO
  private int count;
  // getter / setter for private field (count)
  public int getCount() {
    return count;
  }
  public void setCount(int c) {
    this.count = c;
  }
}
public static class ComplexNestedClass {
  public Integer someNumber;
  public float someFloat;
  public Tuple3<Long, Long, String> word;
  public IntWritable hadoopCitizen;
}
{% endhighlight %}

这些是以上示例代码的有效字段表达式：

- `"count"`：`WC`类中的count 字段。
- `"complex"`：递归地选择POJO类型ComplexNestedClass里的所有字段。 
- `"complex.word.f2"：`选择嵌套的 `Tuple3`类型的最后一个字段。
- `"complex.hadoopCitizen"`：选择Hadoop `IntWritable`类型。 

</div>
<div data-lang="scala" markdown="1">

下面的例子中，我们有一个对象`WC`，它有 "word" 和"count"两个字段，如果根据字段`word`进行分组，我们可以直接把它的名字传入`keyBy()`函数中。{% highlight java %}
// some ordinary POJO (Plain old Java Object)
class WC(var word: String, var count: Int) {
  def this() { this("", 0L) }
}
val words: DataStream[WC] = // [...]
val wordCounts = words.keyBy("word").window(/*window specification*/)

// or, as a case class, which is less typing
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val wordCounts = words.keyBy("word").window(/*window specification*/)
{% endhighlight %}

**字段表达式分析：**

- 按字段名称选择POJO字段。例如，`"user"`指的是POJO类型的“用户”字段。
- 按1开始递增的字段名称或按0开始递增的字段索引来选择元组字段。例如，`"_1"`和`"5"`分别指定的是Scala Tuple类型的第一个和第六个字段。
- 您可以选择POJO和元组中的嵌套字段。例如，`"user.zip"`指定为一个POJO的"zip" 字段，而这个POJO作为"user"字段存储在另一个POJO。支持POJO和元组的任意嵌套和混合，例如`"_2.user.zip"` o或`"user._4.1.zip"。

**字段表达式示例：**

{% highlight scala %}
class WC(var complex: ComplexNestedClass, var count: Int) {
  def this() { this(null, 0) }
}

class ComplexNestedClass(
    var someNumber: Int,
    someFloat: Float,
    word: (Long, Long, String),
    hadoopCitizen: IntWritable) {
  def this() { this(0, 0, (0, 0, ""), new IntWritable(0)) }
}
{% endhighlight %}

这些是以上示例代码的有效字段表达式：

- `"count"`：`WC`类中的count字段。
- `"complex"`：递归地选择POJO类型ComplexNestedClass里的所有字段。 
- `"complex.word._3"：`选择嵌套的 `Tuple3`类型的最后一个字段。
- `"complex.hadoopCitizen"`：选择Hadoop `IntWritable`类型。 

</div>
</div>

### 使用键选择器方法来定义键
{:.no_toc}

另一种定义键的方法是“键选择器”功能。键选择器函数将单个元素作为输入并返回该元素的键。键可以是任何类型，并且可以从任意计算中派生。 

下面的例子展示了一个简单的返回对象字段的键选择器函数：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// some ordinary POJO
public class WC {public String word; public int count;}
DataStream<WC> words = // [...]
KeyedStream<WC> kyed = words
  .keyBy(new KeySelector<WC, String>() {
     public String getKey(WC wc) { return wc.word; }
   });
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// some ordinary case class
case class WC(word: String, count: Int)
val words: DataStream[WC] = // [...]
val keyed = words.keyBy( _.word )
{% endhighlight %}
</div>
</div>

{% top %}

指定转换函数
--------------------------

大多数转换需要用户定义方法。本节列出了如何使用不同的方式来指定它们。 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

#### 实现一个接口

最基本的方法是实现提供的接口之一： 

{% highlight java %}
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
data.map(new MyMapFunction());
{% endhighlight %}

#### 匿名类

您可以将一个函数作为匿名类传递： 
{% highlight java %}
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

#### Java 8 Lambda表达式

Flink还支持Java API中的Java 8 Lambda表达式.请参阅完整的[Java 8指南]({{ site.baseurl }}/dev/java8.html)

{% highlight java %}
data.filter(s -> s.startsWith("http://"));
{% endhighlight %}

{% highlight java %}
data.reduce((i1,i2) -> i1 + i2);
{% endhighlight %}

#### Rich functions

所有需要用户定义函数的转换都可以被*rich* function取代。例如，替换如下方法

{% highlight java %}
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

你可以写为

{% highlight java %}
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

并像往常一样将函数传递给一个`map`转换函数：

{% highlight java %}
data.map(new MyMapFunction());
{% endhighlight %}

Rich functions也可以定义为匿名类 ：

{% highlight java %}
data.map (new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">


#### Lambda Functions

正如在前面的例子中已经看到的那样，所有操作都接受lambda函数来描述操作： 

{% highlight scala %}
val data: DataSet[String] = // [...]
data.filter { _.startsWith("http://") }
{% endhighlight %}

{% highlight scala %}
val data: DataSet[Int] = // [...]
data.reduce { (i1,i2) => i1 + i2 }
// or
data.reduce { _ + _ }
{% endhighlight %}

#### Rich functions

所有lambda函数的转换都可以被*rich* function取代。例如，替换如下方法

{% highlight scala %}
data.map { x => x.toInt }
{% endhighlight %}

你可以这样写

{% highlight scala %}
class MyMapFunction extends RichMapFunction[String, Int] {
  def map(in: String):Int = { in.toInt }
})
{% endhighlight %}

并将函数传递给一个`map`转换： 

{% highlight scala %}
data.map(new MyMapFunction())
{% endhighlight %}

Rich functions也可以定义为匿名类：

{% highlight scala %}
data.map (new RichMapFunction[String, Int] {
  def map(in: String):Int = { in.toInt }
})
{% endhighlight %}
</div>

</div>

Rich functions的提供，除了用户定义的函数（map.reduce等），还有四个方法：`open`, `close`, `getRuntimeContext`和`setRuntimeContext`。这些对参数化函数（请参阅[参数传递给函数]({{ site.baseurl }}/dev/batch/index.html#passing-parameters-to-functions) ），创建和终止本地状态，访问广播变量 （请参阅[广播变量]({{ site.baseurl }}/dev/batch/index.html#broadcast-variables)），以及访问运行时信息（如累加器和计数器）（请参阅[计数器和累加器](#accumulators--counters)）以及迭代（请参阅[迭代]({{ site.baseurl }}/dev/batch/iterations.html)）  都很有用 。

{% top %}

支持的数据类型
--------------------

Flink对可以存在于DataSet或DataStream中的元素的类型有一些限制。原因是系统需要分析类型以确定有效的执行策略。 

有六种不同类型的数据类型： 

1. **Java 元组** 和**Scala 样本类**
2. **Java POJOs**
3. **原始类型**
4. **常规类型**
5. **值**
6. **Hadoop Writables**
7. **特殊类型**

#### Tuples and Case Classes

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

元组是包含固定数量的各种类型的字段的复合类型。Java API提供从`Tuple1`到`Tuple25`的类。。元组的每个字段都可以是包含更多元组的任意Flink类型，从而生成嵌套元组。可以直接使用该字段的名称`tuple.f4`或使用泛型getter方法 来访问元组的字段`tuple.getField(int position)`。字段索引从0开始。请注意，这与Scala元组形成对比，但它与Java的一般索引更加一致。

{% highlight java %}
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public Integer map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});

wordCounts.keyBy(0); // also valid .keyBy("f0")


{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

Scala样本类（以及作为样本类的特例的Scala元组）是包含固定数量的各种类型字段的复合类型。元组字段通过它们的偏移量名称寻址，例如`_1`第一个字段。大小写类字段按名称访问。 

{% highlight scala %}
case class WordCount(word: String, count: Int)
val input = env.fromElements(
    WordCount("hello", 1),
    WordCount("world", 2)) // Case Class Data Set

input.keyBy("word")// key by field expression "word"

val input2 = env.fromElements(("hello", 1), ("world", 2)) // Tuple2 Data Set

input2.keyBy(0, 1) // key by field positions 0 and 1
{% endhighlight %}

</div>
</div>

#### POJOS

如果满足以下要求，Java和Scala类将被Flink视为特殊的POJO数据类型： 

- 类必须是公开的。
- 它必须有一个没有参数的公开的构造函数（默认构造函数）。
- 所有字段都是公开的，或者必须通过getter和setter函数访问。对于称为`foo`getter和setter方法的字段必须命名`getFoo()`和`setFoo()`。
- Flink必须支持字段的类型。目前，Flink使用[Avro](http://avro.apache.org)序列化任意对象（如`Date`）。

Flink分析POJO类型的结构，即它了解POJO的字段。因此POJO类型比一般类型更易于使用。而且，Flink可以比普通类型更有效地处理POJO。

以下示例显示了一个包含两个公用字段的简单POJO。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
public class WordWithCount {

    public String word;
    public int count;
    
    public WordWithCount() {}
    
    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}

DataStream<WordWithCount> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));

wordCounts.keyBy("word"); // key by field expression "word"

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
class WordWithCount(var word: String, var count: Int) {
    def this() {
      this(null, -1)
    }
}

val input = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2)) // Case Class Data Set

input.keyBy("word")// key by field expression "word"

{% endhighlight %}
</div>
</div>

#### 原始类型

Flink支持所有Java和Scala的原始类型，如`Integer`，`String`和`Double`。 

#### 基本类型

Flink支持大多数Java和Scala类（API和自定义）。但是限制包含无法序列化的字段的类，如文件指针，I / O流或其他本地资源。遵循Java Beans约定的类一般运行良好。

所有未被识别为POJO类型的类（见上面的POJO要求）都由Flink作为一般类类型来处理。Flink将这些数据类型视为黑盒子，并且无法访问其内容（即进行高效排序）。常规类型使用序列化框架[Kryo](https://github.com/EsotericSoftware/kryo)进行反序列化。

#### 值

*值*类型手动描述它们的序列化和反序列化。它们不是通过通用序列化框架，而是通过实现`org.apache.flinktypes.Value`接口的`read`和`write`方法来为这些操作提供自定义代码。当通用序列化非常低效时，使用值类型是合理的。例如，一个数据类型，它实现一个稀疏的元素向量作为数组。知道数组大部分为零，可以对非零元素使用特殊编码，而通用序列化将简单地写入所有数组元素。

该`org.apache.flinktypes.CopyableValue`接口以类似的方式支持手动内部克隆逻辑。

Flink带有与基本数据类型相对应的预定义值类型。（`ByteValue`，`ShortValue`，`IntValue`，`LongValue`，`FloatValue`，`DoubleValue`，`StringValue`，`CharValue`， `BooleanValue`）。这些Value类型作为基本数据类型的可变变体：它们的值可以被改变，允许程序员重用对象并从垃圾收集器中释放压力。


#### Hadoop Writables

您可以使用实现`org.apache.hadoop.Writable`接口的类型。在`write()`and `readFields()`方法中定义的序列化逻辑将用于序列化。 

#### Special Types

您可以使用特殊类型，包括Scala的`Either`，`Option`和`Try`。Java API有它自己的自定义实现`Either`。与Scala的`Either`类似，它代表了*Left*或*Right*两种可能类型的值。 `Either`可用于错误处理或需要输出两种不同类型记录的操作。 

#### 类型擦除和类型推断

*注意：本节仅与Java相关。*

Java编译器在编译后会抛弃大部分的泛型类型信息。这被称为Java中的*类型擦除*。这意味着在运行时，对象的一个实例不再知道它的泛型类型。例如，JVM的实例`DataStream<String>`和`DataStream<Long>`看起来相同。

当Flink准备要执行的程序时（当程序的main方法被调用时），Flink需要类型信息。Flink Java API会尝试重建以各种方式丢弃的类型信息，并将其明确存储在数据集和运算符中。您可以通过检索类型`DataStream.getType()`。该方法返回一个实例`TypeInformation`，这是Flink表示类型的内部方式。

类型推断有其局限性，在某些情况下需要程序员的“合作”。这方面的例子是从集合中创建数据集的方法，例如`ExecutionEnvironment.fromCollection(),`可以传递描述类型的参数的地方。但是通用函数`MapFunction<I, O>`也可能需要额外的类型信息。

{% gh_link /flink-core/src/main/java/org/apache/flink/api/java/typeutils/ResultTypeQueryable.java "ResultTypeQueryable" %} 接口可以通过输入格式和功能来实现明确地告诉API他们的返回类型。函数被调用的*输入类型*通常可以由前面的操作的结果类型推断出来。

{% top %}

累加器和计数器
---------------------------

累加器是一个简单的结构，具有**添加操作**和**最终累计结果**，在作业结束后可用。

最直接的累加器是一个**计数器**：您可以使用该`Accumulator.add(V value)`方法增加 **计数器**。在工作结束时，Flink将总结（合并）所有部分结果并将结果发送给客户端。在调试过程中或者您很快想要了解有关数据的更多信息时，累加器很有用。

Flink目前拥有以下**内置累加器**。它们中的每一个都实现了 [累加器](https://github.com/apache/flink/blob/master//flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java) 接口。

- [**IntCounter**](https://github.com/apache/flink/blob/master//flink-core/src/main/java/org/apache/flink/api/common/accumulators/IntCounter.java)， [**LongCounter**](https://github.com/apache/flink/blob/master//flink-core/src/main/java/org/apache/flink/api/common/accumulators/LongCounter.java) 和[ **DoubleCounter**](https://github.com/apache/flink/blob/master//flink-core/src/main/java/org/apache/flink/api/common/accumulators/DoubleCounter.java)：请参阅下面的示例使用计数器。
- [**直方图**](https://github.com/apache/flink/blob/master//flink-core/src/main/java/org/apache/flink/api/common/accumulators/Histogram.java)：离散数量箱的直方图实现。内部它只是一个从Integer到Integer的映射。您可以使用它来计算值的分布，例如字数统计程序的每行字数分布。

**如何使用累加器：**

首先，您必须在要使用它的用户定义的转换函数中创建一个累加器对象（这里是一个计数器）。

{% highlight java %}
private IntCounter numLines = new IntCounter();
{% endhighlight %}

其次，您必须注册累加器对象，通常使用*rich* function的`open()`方法 。在这里你也定义了名字。 

{% highlight java %}
getRuntimeContext().addAccumulator("num-lines", this.numLines);
{% endhighlight %}

您现在可以在操作中的任何位置使用累加器，包括在`open()`和 `close()`方法中。 

{% highlight java %}
this.numLines.add(1);
{% endhighlight %}

总体结果将存储在`JobExecutionResult，即`从运行环境的`execute()`方法返回的对象中（当前仅当执行等待作业完成时才起作用）。 

{% highlight java %}
myJobExecutionResult.getAccumulatorResult("num-lines")
{% endhighlight %}

所有累加器在每个作业共享一个名称空间。因此，您可以在工作的不同操作功能中使用相同的累加器。Flink将内部合并所有具有相同名称的累加器。

关于累加器和迭代的说明：目前累加器的结果只有在整个工作结束后才可用。我们还计划在下一次迭代中提供前一次迭代的结果。您可以使用 [聚合器](https://github.com/apache/flink/blob/master//flink-java/src/main/java/org/apache/flink/api/java/operators/IterativeDataSet.java#L98) 来计算每次迭代统计数据，并基于此类统计数据的迭代终止。

__自定义累加器:__

要实现自己的累加器，只需编写累加器接口的实现。如果您认为您的自定义累加器应与Flink合并到一起，请随时pull request。

您可以选择实现 [Accumulator](https://github.com/apache/flink/blob/master//flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java) 或[SimpleAccumulator](https://github.com/apache/flink/blob/master//flink-core/src/main/java/org/apache/flink/api/common/accumulators/SimpleAccumulator.java)。 

`Accumulator<V,R>`是最灵活的：它定义了`V`要添加的值的类型`R`，以及最终结果的结果类型。例如，对于直方图，`V`是数字并且`R`是直方图。`SimpleAccumulator`适用于两种类型相同的情况，例如柜台。 

{% top %}

