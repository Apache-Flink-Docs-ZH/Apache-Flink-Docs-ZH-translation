---
title: "Flink DataStream API Programming Guide"
nav-title: Streaming (DataStream API)
nav-id: streaming
nav-parent_id: dev
nav-show_overview: true
nav-pos: 10
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

Flink中的DataStream程序是对数据流进行转换（例如，过滤、更新状态、定义窗口、聚合）的常用方式。数据流起于各种sources（例如，消息队列，socket流，文件）。通过sinks返回结果，例如将数据写入文件或标准输出（例如命令行终端）。Flink程序可以运行在各种上下文环境中，独立或嵌入其他程序中。
执行过程可能发生在本地JVM或在由许多机器组成的集群上。

关于Flink API的基本概念介绍请参阅[基本概念]({{ site.baseurl }}/dev/api_concepts.html)。

为了创建你的Flink DataStream程序，我们鼓励你从[解构Flink程序]({{ site.baseurl }}/dev/api_concepts.html#anatomy-of-a-flink-program)
开始，并逐渐添加你自己的[transformations](#datastream-transformations)。本节其余部分作为附加操作和高级功能的参考。


* This will be replaced by the TOC
{:toc}


示例程序
---------------

以下是基于流式窗口进行word count的一个完整可运行的程序示例，它从网络socket中以5秒的窗口统计单词个数。你可以复制并粘贴代码用以本地运行。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(0)
                .timeWindow(Time.seconds(5))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time

object WindowWordCount {
  def main(args: Array[String]) {

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val text = env.socketTextStream("localhost", 9999)

    val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
      .map { (_, 1) }
      .keyBy(0)
      .timeWindow(Time.seconds(5))
      .sum(1)

    counts.print

    env.execute("Window Stream WordCount")
  }
}
{% endhighlight %}
</div>

</div>

要运行示例程序，首先在终端启动netcat作为输入流：

~~~bash
nc -lk 9999
~~~

然后输入一些单词，回车换行输入新一行的单词。这些输入将作为示例程序的输入。如果要使得某个单词的计数大于1，请在5秒钟内重复输入相同的字词（如果5秒钟输入相同单词对你来说太快，请把示例程序中的窗口大小从5秒调大）。

{% top %}

DataStream Transformations
--------------------------

数据的Transformations可以将一个或多个DataStream转换为一个新的DataStream。程序可以将多种Transformations组合成复杂的拓扑结构。

本节对所有可用Transformations进行详细说明。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Transformation</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
          <td><strong>Map</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>读入一个元素，返回转换后的一个元素。一个把输入流转换中的数值翻倍的map function：</p>
    {% highlight java %}
DataStream<Integer> dataStream = //...
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
});
    {% endhighlight %}
          </td>
        </tr>

        <tr>
          <td><strong>FlatMap</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>读入一个元素，返回转换后的0个、1个或者多个元素。一个将句子切分成单词的flatmap function：</p>
    {% highlight java %}
dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Filter</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>对读入的每个元素执行boolean函数，并保留返回true的元素。一个过滤掉零值的filter：
            </p>
    {% highlight java %}
dataStream.filter(new FilterFunction<Integer>() {
    @Override
    public boolean filter(Integer value) throws Exception {
        return value != 0;
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>KeyBy</strong><br>DataStream &rarr; KeyedStream</td>
          <td>
            <p>逻辑上将流分区为不相交的分区，每个分区包含相同key的元素。在内部通过hash分区来实现。关于如何指定分区的keys请参阅<a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys</a>。该transformation返回一个KeyedDataStream。</p>
    {% highlight java %}
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
    {% endhighlight %}
            <p>
            <span class="label label-danger">注意</span> 
            这种类型<strong>不能作为key</strong>:
    	    <ol> 
    	    <li> POJO类型，并且依赖于<em>Object.hashCode()</em>的实现，但是未覆写<em>hashCode()</em></li>
    	    <li> 任意类型的数组</li>
    	    </ol>
    	    </p>
          </td>
        </tr>
        <tr>
          <td><strong>Reduce</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>在一个KeyedStream上不断进行reduce操作。将当前元素与上一个reduce后的值进行合并，再返回新合并的值。
                    <br/>
            	<br/>
            一个构造局部求和流的reduce function：</p>
            {% highlight java %}
keyedStream.reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer value1, Integer value2)
    throws Exception {
        return value1 + value2;
    }
});
            {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Fold</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
          <p>在一个KeyedStream上基于初始值不断进行变换操作，将当前值与上一个变换后的值进行变换，再返回新变换的值。
          <br/>
          <br/>
          <p>在序列（1,2,3,4,5）上应用如下的fold function，返回的序列依次是“start-1”，“start-1-2”，“start-1-2-3”, ...：</p>
          {% highlight java %}
DataStream<String> result =
  keyedStream.fold("start", new FoldFunction<Integer, String>() {
    @Override
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
  });
          {% endhighlight %}
          </p>
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>在一个KeyedStream上不断聚合。min和minBy的区别是min返回最小值，而minBy返回在该字段上值为最小值的所有元素（对于max和maxBy相同）。</p>
    {% highlight java %}
keyedStream.sum(0);
keyedStream.sum("key");
keyedStream.min(0);
keyedStream.min("key");
keyedStream.max(0);
keyedStream.max("key");
keyedStream.minBy(0);
keyedStream.minBy("key");
keyedStream.maxBy(0);
keyedStream.maxBy("key");
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window</strong><br>KeyedStream &rarr; WindowedStream</td>
          <td>
            <p>Windows可定义在已分区的KeyedStreams上。Windows会在每个key对应的数据上根据一些特征（例如，在最近5秒内到达的数据）进行分组。
            有关Windows的完整说明请参阅<a href="windows.html">windows</a>。
    {% highlight java %}
dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
    {% endhighlight %}
        </p>
          </td>
        </tr>
        <tr>
          <td><strong>WindowAll</strong><br>DataStream &rarr; AllWindowedStream</td>
          <td>
              <p>Windows可定义在普通DataStream上。Windows根据一些特征（例如，在最近5秒内到达的数据）对所有流事件进行分组。有关Windows的完整说明请参阅<a href="windows.html">windows</a>。</p>
              <p><strong>警告：</strong>在多数情况下，这是<strong>非并行</strong>的的transformation。所有记录将被聚集到运行windowAll操作的一个任务中。</p>
  {% highlight java %}
dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
  {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Apply</strong><br>WindowedStream &rarr; DataStream<br>AllWindowedStream &rarr; DataStream</td>
          <td>
            <p>把窗口作为整体，并在此整体上应用通用函数。以下是手动对窗口全体元素求和的函数。</p>
            <p><strong>注意：</strong>如果你正在使用windowAll transformation，则需要替换为AllWindowFunction。</p>
    {% highlight java %}
windowedStream.apply (new WindowFunction<Tuple2<String,Integer>, Integer, Tuple, Window>() {
    public void apply (Tuple tuple,
            Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});

// applying an AllWindowFunction on non-keyed window stream
allWindowedStream.apply (new AllWindowFunction<Tuple2<String,Integer>, Integer, Window>() {
    public void apply (Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Reduce</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>在窗口上应用一个通用的reduce函数并返回reduced后的值。</p>
    {% highlight java %}
windowedStream.reduce (new ReduceFunction<Tuple2<String,Integer>>() {
    public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2) throws Exception {
        return new Tuple2<String,Integer>(value1.f0, value1.f1 + value2.f1);
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Fold</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>在窗口上应用一个通用的fold函数并返回fold后的值。在序列（1,2,3,4,5）上应用示例函数，将会得到字符串“start-1-2-3-4-5”中：</p>
    {% highlight java %}
windowedStream.fold("start", new FoldFunction<Integer, String>() {
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations on windows</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>聚合窗口的内容。min和minBy的区别是min返回最小值，而minBy返回在该字段上值为最小值的所有元素（对于max和maxBy相同）。</p>
    {% highlight java %}
windowedStream.sum(0);
windowedStream.sum("key");
windowedStream.min(0);
windowedStream.min("key");
windowedStream.max(0);
windowedStream.max("key");
windowedStream.minBy(0);
windowedStream.minBy("key");
windowedStream.maxBy(0);
windowedStream.maxBy("key");
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Union</strong><br>DataStream* &rarr; DataStream</td>
          <td>
            <p>联合（Union）两个或多个数据流，创建一个包含来自所有流的所有元素的新的数据流。注意：如果DataStream和自身联合，那么在结果流中每个元素你会拿到两份。</p>
    {% highlight java %}
dataStream.union(otherStream1, otherStream2, ...);
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Join</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>在给定的key和公共窗口上连接（Join）两个DataStream。</p>
    {% highlight java %}
dataStream.join(otherStream)
    .where(<key selector>).equalTo(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new JoinFunction () {...});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window CoGroup</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>在给定的key和公共窗口上CoGroup两个DataStream。</p>
    {% highlight java %}
dataStream.coGroup(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new CoGroupFunction () {...});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Connect</strong><br>DataStream,DataStream &rarr; ConnectedStreams</td>
          <td>
            <p>“串联”(Connect)两个DataStream并保留各自类型。串联允许两个流之间共享状态。</p>
    {% highlight java %}
DataStream<Integer> someStream = //...
DataStream<String> otherStream = //...

ConnectedStreams<Integer, String> connectedStreams = someStream.connect(otherStream);
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>CoMap, CoFlatMap</strong><br>ConnectedStreams &rarr; DataStream</td>
          <td>
            <p>在一个ConnectedStreams上做类似map和flatMap的操作。</p>
    {% highlight java %}
connectedStreams.map(new CoMapFunction<Integer, String, Boolean>() {
    @Override
    public Boolean map1(Integer value) {
        return true;
    }

    @Override
    public Boolean map2(String value) {
        return false;
    }
});
connectedStreams.flatMap(new CoFlatMapFunction<Integer, String, String>() {

   @Override
   public void flatMap1(Integer value, Collector<String> out) {
       out.collect(value.toString());
   }

   @Override
   public void flatMap2(String value, Collector<String> out) {
       for (String word: value.split(" ")) {
         out.collect(word);
       }
   }
});
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Split</strong><br>DataStream &rarr; SplitStream</td>
          <td>
            <p>
                根据一些标准将流分成两个或更多个流。
                {% highlight java %}
SplitStream<Integer> split = someDataStream.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>();
        if (value % 2 == 0) {
            output.add("even");
        }
        else {
            output.add("odd");
        }
        return output;
    }
});
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Select</strong><br>SplitStream &rarr; DataStream</td>
          <td>
            <p>
                在一个SplitStream上选择一个或多个流。
                {% highlight java %}
SplitStream<Integer> split;
DataStream<Integer> even = split.select("even");
DataStream<Integer> odd = split.select("odd");
DataStream<Integer> all = split.select("even","odd");
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Iterate</strong><br>DataStream &rarr; IterativeStream &rarr; DataStream</td>
          <td>
            <p>
                通过将一个operator的输出重定向到某个先前的operator，在流中创建“反馈”循环。这对于需要不断更新模型的算法特别有用。
                以下代码以流开始，并持续应用迭代体。大于0的元素将回送到反馈通道，其余元素发往下游。相关完整描述请参阅<a href="#iterations">iterations</a>。
                {% highlight java %}
IterativeStream<Long> iteration = initialStream.iterate();
DataStream<Long> iterationBody = iteration.map (/*do something*/);
DataStream<Long> feedback = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Integer value) throws Exception {
        return value > 0;
    }
});
iteration.closeWith(feedback);
DataStream<Long> output = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Integer value) throws Exception {
        return value <= 0;
    }
});
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Extract Timestamps</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>
                从记录中提取时间戳，以便在窗口中使用事件时间语义。请参阅<a href="{{ site.baseurl }}/dev/event_time.html">Event Time</a>。
                {% highlight java %}
stream.assignTimestamps (new TimeStampExtractor() {...});
                {% endhighlight %}
            </p>
          </td>
        </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
          <td><strong>Map</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>读入一个元素，返回转换后的一个元素。一个把输入流转换中的数值翻倍的map function：</p>
    {% highlight scala %}
dataStream.map { x => x * 2 }
    {% endhighlight %}
          </td>
        </tr>

        <tr>
          <td><strong>FlatMap</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>读入一个元素，返回转换后的0个、1个或者多个元素。一个将句子切分成单词的flatmap function：</p>
    {% highlight scala %}
dataStream.flatMap { str => str.split(" ") }
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Filter</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>对读入的每个元素执行boolean函数，并保留返回true的元素。一个过滤掉零值的filter：
            </p>
    {% highlight scala %}
dataStream.filter { _ != 0 }
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>KeyBy</strong><br>DataStream &rarr; KeyedStream</td>
          <td>
            <p>逻辑上将流分区为不相交的分区，每个分区包含相同key的元素。在内部通过hash分区来实现。关于如何指定分区的keys请参阅<a href="{{ site.baseurl }}/dev/api_concepts.html#specifying-keys">keys</a>。该transformation返回一个KeyedDataStream。</p>
    {% highlight scala %}
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Reduce</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>在一个KeyedStream上不断进行reduce操作。将当前元素与上一个reduce后的值进行合并，再返回新合并的值。
                    <br/>
            	<br/>
            一个构造局部求和流的reduce function：</p>
            {% highlight scala %}
keyedStream.reduce { _ + _ }
            {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Fold</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
          <p>在一个KeyedStream上基于初始值不断进行变换操作，将当前值与上一个变换后的值进行变换，再返回新变换的值。
          <br/>
          <br/>
          <p>在序列（1,2,3,4,5）上应用如下的fold function，返回的序列依次是“start-1”，“start-1-2”，“start-1-2-3”, ...：</p>
          {% highlight scala %}
val result: DataStream[String] =
    keyedStream.fold("start")((str, i) => { str + "-" + i })
          {% endhighlight %}
          </p>
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>在一个KeyedStream上不断聚合。min和minBy的区别是min返回最小值，而minBy返回在该字段上值为最小值的所有元素（对于max和maxBy相同）。</p>
    {% highlight scala %}
keyedStream.sum(0)
keyedStream.sum("key")
keyedStream.min(0)
keyedStream.min("key")
keyedStream.max(0)
keyedStream.max("key")
keyedStream.minBy(0)
keyedStream.minBy("key")
keyedStream.maxBy(0)
keyedStream.maxBy("key")
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window</strong><br>KeyedStream &rarr; WindowedStream</td>
          <td>
            <p>Windows可定义在已分区的KeyedStreams上。Windows会在每个key对应的数据上根据一些特征（例如，在最近5秒内到达的数据）进行分组。有关Windows的完整说明请参阅<a href="windows.html">windows</a>。
    {% highlight scala %}
dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))) // Last 5 seconds of data
    {% endhighlight %}
        </p>
          </td>
        </tr>
        <tr>
          <td><strong>WindowAll</strong><br>DataStream &rarr; AllWindowedStream</td>
          <td>
              <p>Windows可定义在普通DataStream上。Windows根据一些特征（例如，在最近5秒内到达的数据）对所有流事件进行分组。有关Windows的完整说明请参阅<a href="windows.html">windows</a>。
              <p><strong>警告：</strong>在多数情况下，这是<strong>非并行</strong>的transformation。所有记录将被聚集到运行windowAll操作的一个任务中。</p>
  {% highlight scala %}
dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5))) // Last 5 seconds of data
  {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Apply</strong><br>WindowedStream &rarr; DataStream<br>AllWindowedStream &rarr; DataStream</td>
          <td>
            <p>把窗口作为整体，并在此整体上应用通用函数。以下是手动对窗口全体元素求和的函数。</p>
            <p><strong>注意：</strong>如果你正在使用windowAll transformation，则需要替换为AllWindowFunction</p>
    {% highlight scala %}
windowedStream.apply { WindowFunction }

// applying an AllWindowFunction on non-keyed window stream
allWindowedStream.apply { AllWindowFunction }

    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Reduce</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>在窗口上应用一个通用的reduce函数并返回reduced后的值。</p>
    {% highlight scala %}
windowedStream.reduce { _ + _ }
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Fold</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>在窗口上应用一个通用的fold函数并返回fold后的值。在序列（1,2,3,4,5）上应用示例函数，将会得到字符串“start-1-2-3-4-5”中：</p>
          {% highlight scala %}
val result: DataStream[String] =
    windowedStream.fold("start", (str, i) => { str + "-" + i })
          {% endhighlight %}
          </td>
	</tr>
        <tr>
          <td><strong>Aggregations on windows</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>聚合窗口的内容。min和minBy的区别是min返回最小值，而minBy返回在该字段上值为最小值的所有元素（对于max和maxBy相同）。</p>
    {% highlight scala %}
windowedStream.sum(0)
windowedStream.sum("key")
windowedStream.min(0)
windowedStream.min("key")
windowedStream.max(0)
windowedStream.max("key")
windowedStream.minBy(0)
windowedStream.minBy("key")
windowedStream.maxBy(0)
windowedStream.maxBy("key")
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Union</strong><br>DataStream* &rarr; DataStream</td>
          <td>
            <p>联合（Union）两个或多个数据流，创建一个包含来自所有流的所有元素的新的数据流。注意：如果DataStream和自身联合，那么在结果流中每个元素你会拿到两份。</p>
    {% highlight scala %}
dataStream.union(otherStream1, otherStream2, ...)
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Join</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>在给定的key和公共窗口上连接（Join）两个DataStream。</p>
    {% highlight scala %}
dataStream.join(otherStream)
    .where(<key selector>).equalTo(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply { ... }
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window CoGroup</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>在给定的key和公共窗口上CoGroup两个DataStream。</p>
    {% highlight scala %}
dataStream.coGroup(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply {}
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Connect</strong><br>DataStream,DataStream &rarr; ConnectedStreams</td>
          <td>
            <p>“串联”(Connect)两个DataStream并保留各自类型。串联允许两个流之间共享状态。</p>
    {% highlight scala %}
someStream : DataStream[Int] = ...
otherStream : DataStream[String] = ...

val connectedStreams = someStream.connect(otherStream)
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>CoMap, CoFlatMap</strong><br>ConnectedStreams &rarr; DataStream</td>
          <td>
            <p>在一个ConnectedStreams上做类似map和flatMap的操作。</p>
    {% highlight scala %}
connectedStreams.map(
    (_ : Int) => true,
    (_ : String) => false
)
connectedStreams.flatMap(
    (_ : Int) => true,
    (_ : String) => false
)
    {% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Split</strong><br>DataStream &rarr; SplitStream</td>
          <td>
            <p>
                根据一些标准将流分成两个或更多个流。
                {% highlight scala %}
val split = someDataStream.split(
  (num: Int) =>
    (num % 2) match {
      case 0 => List("even")
      case 1 => List("odd")
    }
)
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Select</strong><br>SplitStream &rarr; DataStream</td>
          <td>
            <p>
                在一个SplitStream上选择一个或多个流。
                {% highlight scala %}

val even = split select "even"
val odd = split select "odd"
val all = split.select("even","odd")
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Iterate</strong><br>DataStream &rarr; IterativeStream  &rarr; DataStream</td>
          <td>
            <p>
                通过将一个operator的输出重定向到某个先前的operator，在流中创建“反馈”循环。这对于需要不断更新模型的算法特别有用。以下代码以流开始，并持续应用迭代体。大于0的元素将回送到反馈通道，其余元素发往下游。相关完整描述请参阅<a href="#iterations">iterations</a>。
                {% highlight java %}
initialStream.iterate {
  iteration => {
    val iterationBody = iteration.map {/*do something*/}
    (iterationBody.filter(_ > 0), iterationBody.filter(_ <= 0))
  }
}
                {% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Extract Timestamps</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>
                从记录中提取时间戳，以便在窗口中使用事件时间语义。请参阅<a href="{{ site.baseurl }}/apis/streaming/event_time.html">Event Time</a>。
                {% highlight scala %}
stream.assignTimestamps { timestampExtractor }
                {% endhighlight %}
            </p>
          </td>
        </tr>
  </tbody>
</table>

使用如下的匿名模式匹配从元组、case类和集合中抽取信息：
{% highlight scala %}
val data: DataStream[(Int, String, Double)] = // [...]
data.map {
  case (id, name, temperature) => // [...]
}
{% endhighlight %}
未被API原生支持。要使用该特性，请用<a href="scala_api_extensions.html">Scala API extension</a>。


</div>
</div>

以下transformations可用于元组类型的DataStream：


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Project</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>从元组中选择一部分字段子集
{% highlight java %}
DataStream<Tuple3<Integer, Double, String>> in = // [...]
DataStream<Tuple2<String, Integer>> out = in.project(2,0);
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>


### 物理分区

在一个transformation之后，Flink也提供了底层API以允许用户在必要时精确控制流分区，参见如下的函数。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Custom partitioning</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            使用一个用户自定义Partitioner确定每个元素对应的目标task。
            {% highlight java %}
dataStream.partitionCustom(partitioner, "someKey");
dataStream.partitionCustom(partitioner, 0);
            {% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
     <td><strong>Random partitioning</strong><br>DataStream &rarr; DataStream</td>
     <td>
       <p>
            按照均匀分布以随机的方式对元素进行分区。
            {% highlight java %}
dataStream.shuffle();
            {% endhighlight %}
       </p>
     </td>
   </tr>
   <tr>
      <td><strong>Rebalancing (Round-robin partitioning)</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            以round-robin方式对元素进行分区，使得每个分区负载均衡。在数据倾斜的情况下进行性能优化有用。
            {% highlight java %}
dataStream.rebalance();
            {% endhighlight %}
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Rescaling</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            以round-robin方式对元素分区到下游operations。如果你想从source的每个并行实例分散到若干个mappers以负载均衡，但是你不期望rebalacne()那样进行全局负载均衡，这将会有用。这将仅需要本地数据传输，而不是通过网络传输数据，具体取决于其他配置值，例如TaskManager的插槽数。
        </p>
        <p>
            上游operation所发送的元素被分区到下游operation的哪些子集，取决于上游和下游操作的并发度。例如，如果上游operation并发度为2，而下游operation并发度为6，则其中1个上游operation会将元素分发到3个下游operation，另1个上游operation会将元素分发到另外3个下游operation。相反地，如果上游operation并发度为6，而下游operation并发度为2，则其中3个上游operation会将元素分发到1个下游operation，另1个上游operation会将元素分发到另外1个下游operation。
        </p>
        <p>
            在上下游operation的并行度不是彼此的倍数的情况下，下游operation对应的上游的operation输入数量不同。
        </p>
        <p>
            下图可视化了上面例子中说明的对应关系：
        </p>

        <div style="text-align: center">
            <img src="{{ site.baseurl }}/fig/rescale.svg" alt="Checkpoint barriers in data streams" />
            </div>


        <p>
                    {% highlight java %}
dataStream.rescale();
            {% endhighlight %}

        </p>
      </td>
    </tr>
   <tr>
      <td><strong>Broadcasting</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            广播元素到每个分区。
            {% highlight java %}
dataStream.broadcast();
            {% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Custom partitioning</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            使用一个用户自定义Partitioner确定每个元素对应的目标task。
            {% highlight scala %}
dataStream.partitionCustom(partitioner, "someKey")
dataStream.partitionCustom(partitioner, 0)
            {% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
     <td><strong>Random partitioning</strong><br>DataStream &rarr; DataStream</td>
     <td>
       <p>
            按照均匀分布以随机的方式对元素进行分区。
            {% highlight scala %}
dataStream.shuffle()
            {% endhighlight %}
       </p>
     </td>
   </tr>
   <tr>
      <td><strong>Rebalancing (Round-robin partitioning)</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            以round-robin方式对元素进行分区，使得每个分区负载均衡。在数据倾斜的情况下进行性能优化有用。
            {% highlight scala %}
dataStream.rebalance()
            {% endhighlight %}
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Rescaling</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            以round-robin方式对元素分区到下游operations。如果你想从source的每个并行实例分散到若干个mappers以负载均衡，但是你不期望rebalacne()那样进行全局负载均衡，这将会有用。这将仅需要本地数据传输，而不是通过网络传输数据，具体取决于其他配置值，例如TaskManager的插槽数。
        </p>
        <p>
            上游operation所发送的元素被分区到下游operation的哪些子集，取决于上游和下游操作的并发度。例如，如果上游operation并发度为2，而下游operation并发度为6，则其中1个上游operation会将元素分发到3个下游operation，另1个上游operation会将元素分发到另外3个下游operation。相反地，如果上游operation并发度为6，而下游operation并发度为2，则其中3个上游operation会将元素分发到1个下游operation，另1个上游operation会将元素分发到另外1个下游operation。
        </p>
        <p>
            在上下游operation的并行度不是彼此的倍数的情况下，下游operation对应的上游的operation输入数量不同。

        </p>
        </p>
            下图可视化了上面例子中说明的对应关系：
        </p>

        <div style="text-align: center">
            <img src="{{ site.baseurl }}/fig/rescale.svg" alt="Checkpoint barriers in data streams" />
            </div>


        <p>
                    {% highlight java %}
dataStream.rescale()
            {% endhighlight %}

        </p>
      </td>
    </tr>
   <tr>
      <td><strong>Broadcasting</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            广播元素到每个分区。
            {% highlight scala %}
dataStream.broadcast()
            {% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>

### 任务链接（chaining） 和资源组

链接（chaining）两个依次的transformations意味着将它们运行在同一个线程中以获得更好的性能。Flink默认情况下尽可能进行该链接操作（比如两个依次的map transformations），同时Flink根据需要提供API对链接进行细粒度控制：

如果不想在整个Job上进行默认的链接优化，可以设置`StreamExecutionEnvironment.disableOperatorChaining()`。下面的函数可用于更细粒度的控制链接。注意这些函数只能用在一个DataStream transformation之后，因为它们是指向先前的transformation。例如，你可以`someStream.map(...).startNewChain()`, 但是你不能`someStream.startNewChain()`.

Flink中的一个资源组是一个slot，详情请参阅[slots]({{site.baseurl}}/setup/config.html#configuring-taskmanager-processing-slots). 必要时你可以手动隔离不同slots中的operators。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td>Start new chain</td>
      <td>
        <p>从这个operator开始新建一个新的chain。这两个mappers将被链接起来，而filter不会和第一个mapper链接。
{% highlight java %}
someStream.filter(...).map(...).startNewChain().map(...);
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
      <td>Disable chaining</td>
      <td>
        <p>不与这个map operator链接
{% highlight java %}
someStream.map(...).disableChaining();
{% endhighlight %}
        </p>
      </td>
    </tr>
    <tr>
      <td>Set slot sharing group</td>
      <td>
        <p>设置operation的slot共享组。Flink会把相同slot共享组的operation放在同一个slot中，而把没有slot共享组的operation放到其他slot中。这可以用来隔离slot。如果所有输入operation都在相同的slot共享组中，则slot共享组将继承自输入operation。默认slot共享组是“default”，可以通过调用slotSharingGroup（“default”）将operation显式的放入该组。
{% highlight java %}
someStream.filter(...).slotSharingGroup("name");
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td>Start new chain</td>
      <td>
        <p>从这个operator开始新建一个新的chain。这两个mappers将被链接起来，而filter不会和第一个mapper链接。
{% highlight scala %}
someStream.filter(...).map(...).startNewChain().map(...)
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
      <td>Disable chaining</td>
      <td>
        <p>不与这个map operator链接
{% highlight scala %}
someStream.map(...).disableChaining()
{% endhighlight %}
        </p>
      </td>
    </tr>
  <tr>
      <td>Set slot sharing group</td>
      <td>
        <p>设置operation的slot共享组。Flink会把相同slot共享组的operation放在同一个slot中，而把没有slot共享组的operation放到其他slot中。这可以用来隔离slot。如果所有输入operation都在相同的slot共享组中，则slot共享组将继承自输入operation。默认slot共享组是“default”，可以通过调用slotSharingGroup（“default”）将operation显式的放入该组。
{% highlight java %}
someStream.filter(...).slotSharingGroup("name")
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>


{% top %}

数据Sources
------------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

Sources是你的程序读取输入的地方。你可以通过`StreamExecutionEnvironment.addSource(sourceFunction)`将Source添加到你的程序中。Flink提供了若干已经实现好了的source functions，当然你也可以通过实现`SourceFunction`来自定义非并行的source或者实现`ParallelSourceFunction`接口或者扩展`RichParallelSourceFunction`来自定义并行的source，

`StreamExecutionEnvironment`中可以使用以下几个已实现的stream sources：

基于文件：

- `readTextFile(path)` - 读取文本文件，即符合TextInputFormat规范的文件，并将其作为字符串返回。

- `readFile(fileInputFormat, path)` - 根据指定的文件输入格式读取文件（一次）。

- `readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo)` -  这是上面两个方法内部调用的方法。它根据给定的`fileInputFormat`和读取路径读取文件。根据提供的`watchType`，这个source可以定期（每隔`interval`毫秒）监测给定路径的新数据（`FileProcessingMode.PROCESS_CONTINUOUSLY`），或者处理一次路径对应文件的数据并退出（`FileProcessingMode.PROCESS_ONCE`）。你可以通过`pathFilter`进一步排除掉需要处理的文件。

    *实现:*

    在具体实现上，Flink把文件读取过程分为两个子任务，即*目录监控*和*数据读取*。每个子任务都由单独的实体实现。目录监控由单个非并行（并行度为1）的任务执行，而数据读取由并行运行的多个任务执行。后者的并行性等于作业的并行性。单个目录监控任务的作用是扫描目录（根据`watchType`定期扫描或仅扫描一次），查找要处理的文件并把文件分割成*切分片（splits）*，然后将这些切分片分配给下游reader。reader负责读取数据。每个切分片只能由一个reader读取，但一个reader可以逐个读取多个切分片。

    *重要注意：*

    1. 如果`watchType`设置为`FileProcessingMode.PROCESS_CONTINUOUSLY`，则当文件被修改时，其内容将被重新处理。这会打破“exactly-once”语义，因为在文件末尾附加数据将导致其**所有内容**被重新处理。

    2. 如果`watchType`设置为`FileProcessingMode.PROCESS_ONCE`，则source仅扫描路径一次然后退出，而不等待reader完成文件内容的读取。当然reader会继续阅读，直到读取所有的文件内容。关闭source后就不会再有检查点。这可能导致节点故障后的恢复速度较慢，因为该作业将从最后一个检查点恢复读取。

基于 Socket：

- `socketTextStream` - 从socket读取。元素可以用分隔符切分。

基于集合：

- `fromCollection(Collection)` - 从Java的Java.util.Collection创建数据流。集合中的所有元素类型必须相同。

- `fromCollection(Iterator, Class)` - 从一个迭代器中创建数据流。Class指定了该迭代器返回元素的类型。

- `fromElements(T ...)` - 从给定的对象序列中创建数据流。所有对象类型必须相同。

- `fromParallelCollection(SplittableIterator, Class)` - 从一个迭代器中创建并行数据流。Class指定了该迭代器返回元素的类型。

- `generateSequence(from, to)` - 创建一个生成指定区间范围内的数字序列的并行数据流。

自定义：

- `addSource` - 添加一个新的source function。例如，你可以`addSource(new FlinkKafkaConsumer08<>(...))`以从Apache Kafka读取数据。详情参阅 [connectors]({{ site.baseurl }}/dev/connectors/index.html)。

</div>

<div data-lang="scala" markdown="1">

<br />

Sources是你的程序读取输入的地方。你可以通过`StreamExecutionEnvironment.addSource(sourceFunction)`将Source添加到你的程序中。Flink提供了若干已经实现好了的source functions，当然你也可以通过实现`SourceFunction`来自定义非并行的source或者实现`ParallelSourceFunction`接口或者扩展`RichParallelSourceFunction`来自定义并行的source，

`StreamExecutionEnvironment`中可以使用以下几个已实现的stream sources：

基于文件：

- `readTextFile(path)` - 读取文本文件，即符合TextInputFormat规范的文件，并将其作为字符串返回。

- `readFile(fileInputFormat, path)` - 根据指定的文件输入格式读取文件（一次）。

- `readFile(fileInputFormat, path, watchType, interval, pathFilter)` -  这是上面两个方法内部调用的方法。它根据给定的`fileInputFormat`和读取路径读取文件。根据提供的`watchType`，这个source可以定期（每隔`interval`毫秒）监测给定路径的新数据（`FileProcessingMode.PROCESS_CONTINUOUSLY`），或者处理一次路径对应文件的数据并退出（`FileProcessingMode.PROCESS_ONCE`）。你可以通过`pathFilter`进一步排除掉需要处理的文件。

    *实现:*

    在具体实现上，Flink把文件读取过程分为两个子任务，即*目录监控*和*数据读取*。每个子任务都由单独的实体实现。目录监控由单个非并行（并行度为1）的任务执行，而数据读取由并行运行的多个任务执行。后者的并行性等于作业的并行性。单个目录监控任务的作用是扫描目录（根据`watchType`定期扫描或仅扫描一次），查找要处理的文件并把文件分割成*切分片（splits）*，然后将这些切分片分配给下游reader。reader负责读取数据。每个切分片只能由一个reader读取，但一个reader可以逐个读取多个切分片。

    *重要注意：*

    1. 如果`watchType`设置为`FileProcessingMode.PROCESS_CONTINUOUSLY`，则当文件被修改时，其内容将被重新处理。这会打破“exactly-once”语义，因为在文件末尾附加数据将导致其**所有内容**被重新处理。

    2. 如果`watchType`设置为`FileProcessingMode.PROCESS_ONCE`，则source仅扫描路径一次然后退出，而不等待reader完成文件内容的读取。当然reader会继续阅读，直到读取所有的文件内容。关闭source后就不会再有检查点。这可能导致节点故障后的恢复速度较慢，因为该作业将从最后一个检查点恢复读取。

基于 Socket：

- `socketTextStream` - 从socket读取。元素可以用分隔符切分。

Collection-based:

- `fromCollection(Seq)` - 从Java的Java.util.Collection创建数据流。集合中的所有元素类型必须相同。

- `fromCollection(Iterator)` - 从一个迭代器中创建数据流。Class指定了该迭代器返回元素的类型。

- `fromElements(elements: _*)` - 从给定的对象序列中创建数据流。所有对象类型必须相同。

- `fromParallelCollection(SplittableIterator)` - 从一个迭代器中创建并行数据流。Class指定了该迭代器返回元素的类型。

- `generateSequence(from, to)` - 创建一个生成指定区间范围内的数字序列的并行数据流。

自定义：

- `addSource` - 添加一个新的source function。例如，你可以`addSource(new FlinkKafkaConsumer08<>(...))`以从Apache Kafka读取数据。详情参阅[connectors]({{ site.baseurl }}/dev/connectors/)。

</div>
</div>

{% top %}

数据Sinks
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

数据sinks消费DataStream并将其发往文件、socket、外部系统或进行打印。Flink自带多种内置的输出格式，这些都被封装在对DataStream的操作背后：

- `writeAsText()` / `TextOutputFormat` - 将元素以字符串形式写入。字符串
   通过调用每个元素的*toString()*方法获得。

- `writeAsCsv(...)` / `CsvOutputFormat` - 将元组写入逗号分隔的csv文件。行和字段
   分隔符均可配置。每个字段的值来自对象的*toString()*方法。

- `print()` / `printToErr()`  - 打印每个元素的*toString()*值到标准输出/错误输出流。可以配置前缀信息添加到输出，以区分不同*print*的结果。如果并行度大于1，则task id也会添加到输出前缀上。

- `writeUsingOutputFormat()` / `FileOutputFormat` - 自定义文件输出的方法/基类。支持自定义的对象到字节的转换。

- `writeToSocket` - 根据`SerializationSchema`把元素写到socket

- `addSink` - 调用自定义sink function。Flink自带了很多连接其他系统的连接器（connectors）（如
     Apache Kafka），这些连接器都实现了sink function。

</div>
<div data-lang="scala" markdown="1">

<br />

Data sinks消费DataStream并将其发往文件、socket、外部系统或进行打印。Flink自带多种内置的输出格式，这些都被封装在对DataStream的操作背后：

- `writeAsText()` / `TextOutputFormat` - 将元素以字符串形式写入。字符串
   通过调用每个元素的*toString()*方法获得。

- `writeAsCsv(...)` / `CsvOutputFormat` - 将元组写入逗号分隔的csv文件。行和字段
   分隔符均可配置。每个字段的值来自对象的*toString()*方法。

- `print()` / `printToErr()`  - 打印每个元素的*toString()*值到标准输出/错误输出流。可以配置前缀信息添加到输出，以区分不同*print*的结果。如果并行度大于1，则task id也会添加到输出前缀上。

- `writeUsingOutputFormat()` / `FileOutputFormat` - 自定义文件输出的方法/基类。支持自定义的对象到字节的转换。

- `writeToSocket` - 根据`SerializationSchema`把元素写到socket

- `addSink` - 调用自定义sink function。Flink自带了很多连接其他系统的连接器（connectors）（如
     Apache Kafka），这些连接器都实现了sink function。

</div>
</div>

请注意，`DataStream`上的`write*()`方法主要用于调试目的。它们没有参与Flink的检查点机制，这意味着这些function通常都有
at-least-once语义。数据刷新到目标系统取决于OutputFormat的实现。这意味着并非所有发送到OutputFormat的元素都会立即在目标系统中可见。此外，在失败的情况下，这些记录可能会丢失。

为了可靠，在把流写到文件系统时，使用`flink-connector-filesystem`来实现exactly-once。此外，通过`.addSink(...)`方法自定义的实现可以参与Flink的检查点机制以实现exactly-once语义。

{% top %}

迭代（Iterations）
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<br />

迭代流程序实现一个step function并将其嵌入到`IterativeStream`中。由于这样的DataStream程序可能永远不会结束，所以没有最大迭代次数。事实上你需要指定哪一部分的流被反馈到迭代过程，哪个部分通过`split` 或`filter` transformation向下游转发。在这里，我们展示一个使用过滤器的例子。首先，我们定义一个`IterativeStream`

{% highlight java %}
IterativeStream<Integer> iteration = input.iterate();
{% endhighlight %}

然后，我们使用一系列transformations来指定在循环内执行的逻辑（这里示意一个简单的`map` transformation）

{% highlight java %}
DataStream<Integer> iterationBody = iteration.map(/* this is executed many times */);
{% endhighlight %}

要关闭迭代并定义迭代尾部，需要调用`IterativeStream`的`closeWith(feedbackStream)`方法。传给`closeWith` function的DataStream将被反馈给迭代的头部。一种常见的模式是使用filter来分离流中需要反馈的部分和需要继续发往下游的部分。这些filter可以定义“终止”逻辑，以控制元素是流向下游而不是反馈迭代。

{% highlight java %}
iteration.closeWith(iterationBody.filter(/* one part of the stream */));
DataStream<Integer> output = iterationBody.filter(/* some other part of the stream */);
{% endhighlight %}

默认情况下，反馈流的分区将自动设置为与迭代的头部的输入分区相同。用户可以在`closeWith`方法中设置一个可选的boolean标志来覆盖默认行为。

例如，如下程序从一系列整数连续减1，直到它们达到零：

{% highlight java %}
DataStream<Long> someIntegers = env.generateSequence(0, 1000);

IterativeStream<Long> iteration = someIntegers.iterate();

DataStream<Long> minusOne = iteration.map(new MapFunction<Long, Long>() {
  @Override
  public Long map(Long value) throws Exception {
    return value - 1 ;
  }
});

DataStream<Long> stillGreaterThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value > 0);
  }
});

iteration.closeWith(stillGreaterThanZero);

DataStream<Long> lessThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value <= 0);
  }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

<br />

迭代streaming程序实现一个step function并将其嵌入到`IterativeStream`中。由于这样的DataStream程序可能永远不会结束，所以没有最大迭代次数。事实上你需要指定哪一部分的流被反馈到迭代过程，哪个部分通过`split` 或`filter` transformation向下游转发。在这里，我们展示一个迭代的例子，其中主体（计算部分被反复执行）是简单的map transformation，迭代反馈的元素和继续发往下游的元素通过filters进行区分。

{% highlight scala %}
val iteratedStream = someDataStream.iterate(
  iteration => {
    val iterationBody = iteration.map(/* this is executed many times */)
    (tail.filter(/* one part of the stream */), tail.filter(/* some other part of the stream */))
})
{% endhighlight %}


默认情况下，反馈流的分区将自动设置为与迭代的头部的输入分区相同。用户可以在`closeWith`方法中设置一个可选的boolean标志来覆盖默认行为。

例如，如下程序从一系列整数连续减1，直到它们达到零：

{% highlight scala %}
val someIntegers: DataStream[Long] = env.generateSequence(0, 1000)

val iteratedStream = someIntegers.iterate(
  iteration => {
    val minusOne = iteration.map( v => v - 1)
    val stillGreaterThanZero = minusOne.filter (_ > 0)
    val lessThanZero = minusOne.filter(_ <= 0)
    (stillGreaterThanZero, lessThanZero)
  }
)
{% endhighlight %}

</div>
</div>

{% top %}

执行参数
--------------------

`StreamExecutionEnvironment`包含`ExecutionConfig`，它允许为作业运行时进行配置。

更多配置参数请参阅[execution configuration]({{ site.baseurl }}/dev/execution_configuration.html)。下面的参数是DataStream API特有的：

- `enableTimestamps()` / **`disableTimestamps()`**: 如果启用，则从source发出的每一条记录都会附加一个时间戳。
    `areTimestampsEnabled()` 返回当前是否启用该值。

- `setAutoWatermarkInterval(long milliseconds)`: 设置自动发射watermark的间隔。你可以通过`long getAutoWatermarkInterval()`获取当前的发射间隔。

{% top %}

### 容错

[State & Checkpointing]({{ site.baseurl }}/dev/stream/checkpointing.html) 描述了如何开启和配置Flink的checkpointing机制。

### 延迟控制

默认情况下，元素不会逐个传输（这将导致不必要的网络流量），而是被缓冲的。缓冲（实际是在机器之间传输）的大小可以在Flink配置文件中设置。虽然这种方法对于优化吞吐量有好处，但是当输入流不够快时，它可能会导致延迟问题。要控制吞吐量和延迟，你可以在execution environment（或单个operator）上使用`env.setBufferTimeout(timeoutMillis)`来设置缓冲区填满的最大等待时间。如果超过该最大等待时间，即使缓冲区未满，也会被自动发送出去。该最大等待时间默认值为100 ms。

Usage:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment
env.setBufferTimeout(timeoutMillis)

env.genereateSequence(1,10).map(myMap).setBufferTimeout(timeoutMillis)
{% endhighlight %}
</div>
</div>

为了最大化吞吐量，可以设置`setBufferTimeout(-1)`，这样就没有了超时机制，缓冲区只有在满时才会发送出去。为了最小化延迟，可以把超时设置为接近0的值（例如5或10 ms）。应避免将该超时设置为0，因为这样可能导致性能严重下降。

{% top %}

调试
---------

在分布式集群中运行Streaming程序之前，最好确保实现的算法可以正常工作。因此，实施数据分析程序通常是一个渐进的过程：检查结果，调试和改进。

Flink提供了诸多特性来大幅简化数据分析程序的开发：你可以在IDE中进行本地调试，注入测试数据，收集结果数据。本节给出一些如何简化Flink程序开发的指导。

### 本地执行环境

`LocalStreamEnvironment`会在其所在的进程中启动一个Flink引擎. 如果你在IDE中启动LocalEnvironment，你可以在你的代码中设置断点，轻松调试你的程序。

一个LocalEnvironment的创建和使用示例如下：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

DataStream<String> lines = env.addSource(/* some source */);
// build your program

env.execute();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = StreamExecutionEnvironment.createLocalEnvironment()

val lines = env.addSource(/* some source */)
// build your program

env.execute()
{% endhighlight %}
</div>
</div>

### 基于集合的数据Sources

Flink提供了基于Java集合实现的特殊数据sources用于测试。一旦程序通过测试，它的sources和sinks可以方便的替换为从外部系统读写的sources和sinks。

基于集合的数据Sources可以像这样使用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// Create a DataStream from a list of elements
DataStream<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataStream from any Java collection
List<Tuple2<String, Integer>> data = ...
DataStream<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataStream from an Iterator
Iterator<Long> longIt = ...
DataStream<Long> myLongs = env.fromCollection(longIt, Long.class);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.createLocalEnvironment()

// Create a DataStream from a list of elements
val myInts = env.fromElements(1, 2, 3, 4, 5)

// Create a DataStream from any Collection
val data: Seq[(String, Int)] = ...
val myTuples = env.fromCollection(data)

// Create a DataStream from an Iterator
val longIt: Iterator[Long] = ...
val myLongs = env.fromCollection(longIt)
{% endhighlight %}
</div>
</div>

**注意：**目前，集合数据source要求数据类型和迭代器实现`Serializable`。并行度 = 1）。

### 迭代的数据Sink

Flink还提供了一个sink来收集DataStream的测试和调试结果。它可以这样使用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.contrib.streaming.DataStreamUtils

DataStream<Tuple2<String, Integer>> myResult = ...
Iterator<Tuple2<String, Integer>> myOutput = DataStreamUtils.collect(myResult)
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
import org.apache.flink.contrib.streaming.DataStreamUtils
import scala.collection.JavaConverters.asScalaIteratorConverter

val myResult: DataStream[(String, Int)] = ...
val myOutput: Iterator[(String, Int)] = DataStreamUtils.collect(myResult.getJavaStream).asScala
{% endhighlight %}
</div>
</div>

{% top %}
