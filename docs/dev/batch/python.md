---
title: "Python Programming Guide"
is_beta: true
nav-title: Python API
nav-parent_id: batch
nav-pos: 4
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

Flink中的分析程序实现了对数据集的某些操作 (例如，数据过滤，映射，合并，分组)。这些数据最初来源于特定的数据源(例如来自于读文件或数据集合)。操作执行的结果通过数据池以写入数据到(分布式)文件系统或标准输出(例如命令行终端)的形式返回。Flink程序可以运行在不同的环境中，既能够独立运行，也可以嵌入到其他程序中运行。程序可以运行在本地的JVM上，也可以运行在服务器集群中。

为了创建你自己的Flink程序，我们鼓励你从[program skeleton](#https://ci.apache.org/projects/flink/flink-docs-release-1.2/dev/batch/python.html#program-skeleton)（程序框架）开始，并逐渐增加你自己的[transformations](https://ci.apache.org/projects/flink/flink-docs-release-1.2/dev/batch/python.html#transformations)（变化）。以下是更多的用法和高级特性的索引。

* This will be replaced by the TOC
{:toc}

示例程序
---------------

以下程序是一段完整可运行的WordCount示例程序。你可以复制粘贴这些代码并在本地运行。

{% highlight python %}
from flink.plan.Environment import get_environment
from flink.functions.GroupReduceFunction import GroupReduceFunction

class Adder(GroupReduceFunction):
  def reduce(self, iterator, collector):
    count, word = iterator.next()
    count += sum([x[0] for x in iterator])
    collector.collect((count, word))

env = get_environment()
data = env.from_elements("Who's there?",
 "I think I hear them. Stand, ho! Who's there?")

data \
  .flat_map(lambda x, c: [(1, word) for word in x.lower().split()]) \
  .group_by(1) \
  .reduce_group(Adder(), combinable=True) \
  .output()

env.execute(local=True)
{% endhighlight %}

{% top %}

程序框架
----------------

从示例程序可以看出，Flink程序看起来就像普通的python程序一样。每个程序都包含相同的基本组成部分：

1. 获取一个`运行环境`，
2. 加载/创建初始数据，
3. 指定对这些数据的操作，
4. 指定计算结果的存放位置，
5. 运行程序。

接下来，我们将对每个步骤给出概述，更多细节可以参考与之对应的小节。

`Environment`（运行环境）是所有Flink程序的基础。你可以通过调用`Environment`类中的一些静态方法来建立一个环境:

{% highlight python %}
get_environment()
{% endhighlight %}

运行环境可通过多种读文件的方式来指定数据源。如果是简单的按行读取文本文件，你可以采用:

{% highlight python %}
env = get_environment()
text = env.read_text("file:///path/to/file")
{% endhighlight %}

这样，你就获得了可以进行操作（apply transformations）的数据集。关于数据源和输入格式的更多信息，请参考[Data Sources](https://ci.apache.org/projects/flink/flink-docs-release-1.2/dev/batch/python.html#data-sources)。

一旦你获得了一个数据集DataSet，你就可以通过transformations来创建一个新的数据集，并把它写入到文件，再次transform，或者与其他数据集相结合。你可以通过对数据集调用自己个性化定制的函数来进行数据操作。例如，一个类似这样的数据映射操作：

{% highlight python %}
data.map(lambda x: x*2)
{% endhighlight %}

这将会创建一个新的数据集，其中的每个数据都是原来数据集中的2倍。若要获取关于所有transformations的更多信息，及所有数据操作的列表，请参考[Transformations](https://ci.apache.org/projects/flink/flink-docs-release-1.2/dev/batch/python.html#transformations)。

当你需要将所获得的数据集写入到磁盘时，调用下面三种函数的其中一个即可：

{% highlight python %}
data.write_text("<file-path>", WriteMode=Constants.NO_OVERWRITE)
write_csv("<file-path>", line_delimiter='\n', field_delimiter=',', write_mode=Constants.NO_OVERWRITE)
output()
{% endhighlight %}

其中，最后一种方法仅适用于在本机上进行开发/调试，它会将数据集的内容输出到标准输出。（请注意，当函数在集群上运行时，结果将会输出到整个集群节点的标准输出流，即输出到workers的.out文件。）前两种方法，能够将数据集写入到对应的文件中。关于写入到文件的更多信息，请参考 [Data Sinks](#data-sinks)。

当你设计好了程序之后，你需要在环境中执行 `execute` 命令来运行程序。可以选择在本机运行，也可以提交到集群运行，这取决于Flink的创建方式。你可以通过设置 `execute(local=True)` 强制程序在本机运行。


{% top %}

创建项目
---------------

除了搭建好Flink运行环境，就无需进行其他准备工作了。Python包可以从你的Flink版本对应的 /resource 文件夹找到。在执行工作任务时，Flink包，plan包和optional包均可以通过HDFS自动分发。

Python API已经在安装了Python2.7或3.4的Linux/Windows系统上测试过。

默认情况下，Flink通过调用“python”或“python3”来启动python进程，这取决于使用了哪种启动脚本。通过在 flink-conf.yaml 中设置 “python.binary.python[2/3]”对应的值，来设定你所需要的启动方式。

{% top %}

延迟(惰性)求值
---------------

所有的Flink程序都是延迟执行的。当程序的主函数执行时，数据的载入和操作并没有在当时发生。与此相反，每一个被创建出来的操作都被加入到程序的计划中。当程序环境中的某个对象调用了 `execute()` 函数时，这些操作才会被真正的执行。不论该程序是在本地运行还是集群上运行。

延迟求值能够让你建立复杂的程序，并在Flink上以一个整体的计划单元来运行。

{% top %}


数据变换
---------------

数据变换（Data transformations）可以将一个或多个数据集映射为一个新的数据集。程序能够将多种变换结合到一起来进行复杂的整合变换。

该小节将概述各种可以实现的数据变换。[transformations
documentation](dataset_transformations.html)数据变换文档中，有关于所有数据变换和示例的全面介绍。

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description 变换描述</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>Map</strong></td>
      <td>
        <p>输入一个元素，输出一个元素</p>
{% highlight python %}
data.map(lambda x: x * 2)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>输入一个元素，输出0,1，或多个元素 </p>
{% highlight python %}
data.flat_map(
  lambda x,c: [(1,word) for word in line.lower().split() for line in x])
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
        <p> 通过一次函数调用实现并行的分割操作。该函数将分割变换作为一个”迭代器”，并且能够产生任意数量的输出值。每次分割变换的元素数量取决于变换的并行性和之前的操作结果。</p>
{% highlight python %}
data.map_partition(lambda x,c: [value * 2 for value in x])
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>对每一个元素，计算一个布尔表达式的值，保留函数计算结果为true的元素。</p>
{% highlight python %}
data.filter(lambda x: x > 1000)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>通过不断的将两个元素组合为一个，来将一组元素结合为一个单一的元素。这种缩减变换可以应用于整个数据集，也可以应用于已分组的数据集。</p>
{% highlight python %}
data.reduce(lambda x,y : x + y)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>将一组元素缩减为1个或多个元素。缩减分组变换可以被应用于一个完整的数据集，或者一个分组数据集。</p>
{% highlight python %}
class Adder(GroupReduceFunction):
  def reduce(self, iterator, collector):
    count, word = iterator.next()
    count += sum([x[0] for x in iterator)      
    collector.collect((count, word))

data.reduce_group(Adder())
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>对一个数据集包含所有元组的一个域，或者数据集的每个数据组，执行某项built-in操作(求和，求最小值，求最大值)。聚集变换可以被应用于一个完整的数据集，或者一个分组数据集。</p>
{% highlight python %}
# This code finds the sum of all of the values in the first field and the maximum of all of the values in the second field
data.aggregate(Aggregation.Sum, 0).and_agg(Aggregation.Max, 1)

# min(), max(), and sum() syntactic sugar functions are also available
data.sum(0).and_agg(Aggregation.Max, 1)
{% endhighlight %}
      </td>
    </tr>

    </tr>
      <td><strong>Join</strong></td>
      <td>
        J对两个数据集进行联合变换，将得到一个新的数据集，其中包含在两个数据集中拥有相等关键字的所有元素对。也可通过JoinFunction来把成对的元素变为单独的元素。关于join keys的更多信息请查看 <a href="#specifying-keys">keys</a> 。
{% highlight python %}
# In this case tuple fields are used as keys.
# "0" is the join field on the first tuple
# "1" is the join field on the second tuple.
result = input1.join(input2).where(0).equal_to(1)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>CoGroup</strong></td>
      <td>
        <p>The two-dimensional variant of the reduce operation. Groups each input on one or more
        fields and then joins the groups. The transformation function is called per pair of groups.
        See <a href="#specifying-keys">keys</a> on how to define coGroup keys.</p>
{% highlight python %}
data1.co_group(data2).where(0).equal_to(1)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>计算两个输入数据集的笛卡尔乘积(向量叉乘)，得到所有元素对。也可通过CrossFunction实现将一对元素转变为一个单独的元素。</p>
{% highlight python %}
result = data1.cross(data2)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>将两个数据集进行合并。</p>
{% highlight python %}
data.union(data2)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>ZipWithIndex</strong></td>
      <td>
        <p>为数据组中的元素逐个分配连续的索引。了解更多信息，请参考 [Zip Elements Guide](zip_elements_guide.html#zip-with-a-dense-index).</p>
{% highlight python %}
data.zip_with_index()
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

{% top %}


指定Keys
-------------

一些变换（例如Join和CoGroup），需要在进行变换前，为作为输入参数的数据集指定一个关键字，而另一些变换（例如Reduce和GroupReduce），则允许在变换操作之前，对数据集根据某个关键字进行分组。

数据集可通过如下方式分组：

{% highlight python %}
reduced = data \
  .group_by(<define key here>) \
  .reduce_group(<do something>)
{% endhighlight %}

Flink中的数据模型并不是基于键-值对。你无需将数据集整理为keys和values的形式。键是”虚拟的”：它们被定义为在真实数据之上，引导分组操作的函数。

### 为元组定义keys
{:.no_toc}

最简单的情形是对一个数据集中的元组按照一个或多个域进行分组：

{% highlight python %}
reduced = data \
  .group_by(0) \
  .reduce_group(<do something>)
{% endhighlight %}

数据集中的元组被按照第一个域分组。对于接下来的group-reduce函数，输入的数据组中，每个元组的第一个域都有相同的值。

{% highlight python %}
grouped = data \
  .group_by(0,1) \
  .reduce(/*do something*/)
{% endhighlight %}

在上面的例子中，数据集的分组基于第一个和第二个域形成的复合关键字，因此，reduce函数输入数据组中，每个元组两个域的值均相同。

关于嵌套元组需要注意：如果你有一个使用了嵌套元组的数据集，指定 `group_by(<index of tuple>)` 操作，系统将把整个元组作为关键字使用。

{% top %}


向Flink传递函数
--------------------------

一些特定的操作需要采用用户自定义的函数，因此它们都接受lambda表达式和rich functions作为输入参数。

{% highlight python %}
data.filter(lambda x: x > 5)
{% endhighlight %}

{% highlight python %}
class Filter(FilterFunction):
    def filter(self, value):
        return value > 5

data.filter(Filter())
{% endhighlight %}


Rich functions可以将函数作为输入参数，允许使用broadcast-variables（广播变量），能够由init()函数参数化，是复杂函数的一个可考虑的实现方式。它们也是在reduce操作中，定义一个可选的`combine` 函数的唯一方式。

Lambda表达式可以让函数在一行代码上实现，非常便捷。需要注意的是，如果某个操作会返回多个数值，则其使用的lambda表达式应当返回一个迭代器。（所有函数将接收一个collector输入 参数）。

{% top %}

数据类型
----------

Flink的Python API目前仅支持python中的基本数据类型(int,float,bool,string)以及byte arrays。

运行环境对数据类型的支持，包括序列化器serializer，反序列化器deserializer，以及自定义类型的类。
{% highlight python %}
class MyObj(object):
    def __init__(self, i):
        self.value = i


class MySerializer(object):
    def serialize(self, value):
        return struct.pack(">i", value.value)


class MyDeserializer(object):
    def _deserialize(self, read):
        i = struct.unpack(">i", read(4))[0]
        return MyObj(i)


env.register_custom_type(MyObj, MySerializer(), MyDeserializer())
{% endhighlight %}

#### 元组/列表

你可以使用元组（或列表）来表示复杂类型。Python中的元组可以转换为Flink中的Tuple类型，它们包含数量固定的不同类型的域（最多25个）。每个域的元组可以是基本数据类型，也可以是其他的元组类型，从而形成嵌套元组类型。

{% highlight python %}
word_counts = env.from_elements(("hello", 1), ("world",2))

counts = word_counts.map(lambda x: x[1])
{% endhighlight %}

当进行一些要求指定关键字的操作时，例如对数据记录进行分组或配对。通过设定关键字，可以非常便捷地指定元组中各个域的位置。你可以指定多个位置，从而实现复合关键字（更多信息，查阅 [Section Data Transformations](#transformations)).

{% highlight python %}
wordCounts \
    .group_by(0) \
    .reduce(MyReduceFunction())
{% endhighlight %}

{% top %}

数据源
------------

数据源创建了初始的数据集，包括来自文件，以及来自数据接口/集合两种方式。

基于文件的：

- `read_text(path)` – 按行读取文件，并将每一行以String形式返回。
- `read_csv(path, type)` – 解析以逗号（或其他字符）划分数据域的文件。
  返回一个包含若干元组的数据集。支持基本的java数据类型作为字段类型。

基于数据集合的：

- `from_elements(*args)` – 基于一系列数据创建一个数据集，包含所有元素。
- `generate_sequence(from, to)` – 按照指定的间隔，生成一系列数据。


**例子**

{% highlight python %}
env  = get_environment

\# read text file from local files system
localLiens = env.read_text("file:#/path/to/my/textfile")

\# read text file from a HDFS running at nnHost:nnPort
hdfsLines = env.read_text("hdfs://nnHost:nnPort/path/to/my/textfile")

\# read a CSV file with three fields, schema defined using constants defined in flink.plan.Constants
csvInput = env.read_csv("hdfs:///the/CSV/file", (INT, STRING, DOUBLE))

\# create a set from some given elements
values = env.from_elements("Foo", "bar", "foobar", "fubar")

\# generate a number sequence
numbers = env.generate_sequence(1, 10000000)
{% endhighlight %}

{% top %}

数据池
----------

数据池可以接收数据集，并被用来存储或返回它们：

- `write_text()` – 按行以String形式写入数据。可通过对每个数据项调用str()函数获取String。
- `write_csv(...)` – 将元组写入逗号分隔数值文件。行数和数据字段均可配置。每个字段的值可通过对数据项调用str()方法得到。
- `output()`– 在标准输出上打印每个数据项的str()字符串。

一个数据集可以同时作为多个操作的输入数据。程序可以在写入或打印一个数据集的同时，对其进行其他的变换操作。

**例子**

标准数据池相关方法示例如下：

{% highlight scala %}
 write DataSet to a file on the local file system
textData.write_text("file:///my/result/on/localFS")

 write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.write_text("hdfs://nnHost:nnPort/my/result/on/localFS")

 write DataSet to a file and overwrite the file if it exists
textData.write_text("file:///my/result/on/localFS", WriteMode.OVERWRITE)

 tuples as lines with pipe as the separator "a|b|c"
values.write_csv("file:///path/to/the/result/file", line_delimiter="\n", field_delimiter="|")

 this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.write_text("file:///path/to/the/result/file")
{% endhighlight %}

{% top %}

广播变量
-------------------

使用广播变量，能够在使用普通输入参数的基础上，使得一个数据集同时被多个并行的操作所使用。这对于实现辅助数据集，或者是基于数据的参数化法非常有用。这样，数据集就可以以集合的形式被访问。

- **Broadcast**: 注册广播变量，广播数据集可通过调用`with_broadcast_set(DataSet, String)`函数，按照名字注册广播变量
- **Access**：访问广播变量，通过对调用`self.context.get_broadcast_variable(String)`可获取广播变量。


{% highlight python %}
class MapperBcv(MapFunction):
    def map(self, value):
        factor = self.context.get_broadcast_variable("bcv")[0][0]
        return value * factor

# 1. The DataSet to be broadcasted
toBroadcast = env.from_elements(1, 2, 3)
data = env.from_elements("a", "b")

# 2. Broadcast the DataSet
data.map(MapperBcv()).with_broadcast_set("bcv", toBroadcast)
{% endhighlight %}

确保在进行广播变量的注册和访问时，应当采用相同的名字（示例中的`bcv`）。

**提示**：由于广播变量的内容被保存在每个节点的内部存储中，不适合包含过多内容。一些简单的参数，例如标量值，可简单地通过参数化rich function来实现。

{% top %}

并行执行
------------------

This section describes how the parallel execution of programs can be configured in Flink. A Flink
program consists of multiple tasks (operators, data sources, and sinks). A task is split into
several parallel instances for execution and each parallel instance processes a subset of the task's
input data. The number of parallel instances of a task is called its *parallelism* or *degree of
parallelism (DOP)*.

The degree of parallelism of a task can be specified in Flink on different levels.

该章节将描述如何在Flink中配置程序的并行执行。一个Flink程序可以包含多个任务（操作，数据源和数据池）。一个任务可以被划分为多个可并行运行的部分，每个部分处理输入数据的一个子集。并行运行的实例数量被称作它的 *并行性* 或 *并行度*（*degree of parallelism (DOP)*）。

在Flink中可以为任务指定不同等级的并行度。

### 运行环境级

Flink程序可在一个运行环境 [execution environment](#program-skeleton)的上下文中运行。一个运行环境为其中运行的所有操作，数据源和数据池定义了一个默认的并行度。运行环境的并行度可通过对某个操作的并行度进行配置来修改。

一个运行环境的并行度可通过调用 `set_parallelism()` 方法来指定。例如，为了将 [WordCount](#example-program) 示例程序中的所有操作，数据源和数据池的并行度设置为 `3` ，可以通过如下方式设置运行环境的默认并行度：

{% highlight python %}
env = get_environment()
env.set_parallelism(3)

text.flat_map(lambda x,c: x.lower().split()) \
    .group_by(1) \
    .reduce_group(Adder(), combinable=True) \
    .output()

env.execute()
{% endhighlight %}

### 系统级

通过设置位于 `./conf/flink-conf.yaml` 文件的 `parallelism.default` 属性，改变系统级的默认并行度，可设置所有运行环境的默认并行度。具体细节可查阅 [Configuration]({{ site.baseurl }}/setup/config.html) 文档。

{% top %}

执行计划
---------------

To run the plan with Flink, go to your Flink distribution, and run the pyflink.sh script from the /bin folder.
The script containing the plan has to be passed as the first argument, followed by a number of additional python
packages, and finally, separated by - additional arguments that will be fed to the script.

为了在Flink中运行计划任务，到Flink目录下，运行`/bin`文件夹下的 `pyflink.sh` 脚本。包含计划任务的脚本应当作为第一个输入参数，其后可添加一些另外的python包，最后，在“-”之后，输入其他附加参数。

{% highlight python %}
./bin/pyflink.sh <Script>[ <pathToPackage1>[ <pathToPackageX]][ - <param1>[ <paramX>]]
{% endhighlight %}

{% top %}
