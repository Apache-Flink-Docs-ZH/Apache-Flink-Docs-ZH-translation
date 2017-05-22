---
title: "Scala API Extensions"
nav-parent_id: api-concepts
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

为了保持 Scala 和 Java API 的一致性，对于批处理和流处理从标准的API中忽视了一些在 Scala 中允许的高级表达方式。


如果你想体验全部的 Scala 表达功能，你可以选择通过隐式转化来加强 Scala API。



你可以通过简单的导入 DataSet API 来使用所有可用的扩展

{% highlight scala %}
import org.apache.flink.api.scala.extensions._
{% endhighlight %}

或者导入 DataStream API。

{% highlight scala %}
import org.apache.flink.streaming.api.scala.extensions._
{% endhighlight %}

作为选择，你也可以导入私有扩展a-là-carte 来使用那些你更喜欢的。

## 偏函数

通常，数据集和数据流 API 不接受匿名形式的函数去解构元组，例如类或集合，像下面这样：

{% highlight scala %}
val data: DataSet[(Int, String, Double)] = // [...]
data.map {
  case (id, name, temperature) => // [...]
  // 上面一行会引起下面的编译错误:
  // "匿名函数的参数类型必须完全可知. (SLS 8.5)"
}
{% endhighlight %}

这个扩展介绍了在数据集和数据流Scala 的扩展API 中有一对一关系的新的方法。这些授权方法不支持匿名形式的匹配函数。

#### DataSet API

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Method</th>
      <th class="text-left" style="width: 20%">Original</th>
      <th class="text-center">Example</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>mapWith</strong></td>
      <td><strong>map (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.mapWith {
  case (_, value) => value.toString
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>mapPartitionWith</strong></td>
      <td><strong>mapPartition (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.mapPartitionWith {
  case head #:: _ => head
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>flatMapWith</strong></td>
      <td><strong>flatMap (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.flatMapWith {
  case (_, name, visitTimes) => visitTimes.map(name -> _)
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>filterWith</strong></td>
      <td><strong>filter (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.filterWith {
  case Train(_, isOnTime) => isOnTime
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>reduceWith</strong></td>
      <td><strong>reduce (DataSet, GroupedDataSet)</strong></td>
      <td>
{% highlight scala %}
data.reduceWith {
  case ((_, amount1), (_, amount2)) => amount1 + amount2
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>reduceGroupWith</strong></td>
      <td><strong>reduceGroup (GroupedDataSet)</strong></td>
      <td>
{% highlight scala %}
data.reduceGroupWith {
  case id #:: value #:: _ => id -> value
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>groupingBy</strong></td>
      <td><strong>groupBy (DataSet)</strong></td>
      <td>
{% highlight scala %}
data.groupingBy {
  case (id, _, _) => id
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>sortGroupWith</strong></td>
      <td><strong>sortGroup (GroupedDataSet)</strong></td>
      <td>
{% highlight scala %}
grouped.sortGroupWith(Order.ASCENDING) {
  case House(_, value) => value
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>combineGroupWith</strong></td>
      <td><strong>combineGroup (GroupedDataSet)</strong></td>
      <td>
{% highlight scala %}
grouped.combineGroupWith {
  case header #:: amounts => amounts.sum
}
{% endhighlight %}
      </td>
    <tr>
      <td><strong>projecting</strong></td>
      <td><strong>apply (JoinDataSet, CrossDataSet)</strong></td>
      <td>
{% highlight scala %}
data1.join(data2).
  whereClause(case (pk, _) => pk).
  isEqualTo(case (_, fk) => fk).
  projecting {
    case ((pk, tx), (products, fk)) => tx -> products
  }

data1.cross(data2).projecting {
  case ((a, _), (_, b) => a -> b
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>projecting</strong></td>
      <td><strong>apply (CoGroupDataSet)</strong></td>
      <td>
{% highlight scala %}
data1.coGroup(data2).
  whereClause(case (pk, _) => pk).
  isEqualTo(case (_, fk) => fk).
  projecting {
    case (head1 #:: _, head2 #:: _) => head1 -> head2
  }
}
{% endhighlight %}
      </td>
    </tr>
    </tr>
  </tbody>
</table>

#### DataStream API

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Method</th>
      <th class="text-left" style="width: 20%">Original</th>
      <th class="text-center">Example</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>mapWith</strong></td>
      <td><strong>map (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.mapWith {
  case (_, value) => value.toString
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>mapPartitionWith</strong></td>
      <td><strong>mapPartition (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.mapPartitionWith {
  case head #:: _ => head
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>flatMapWith</strong></td>
      <td><strong>flatMap (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.flatMapWith {
  case (_, name, visits) => visits.map(name -> _)
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>filterWith</strong></td>
      <td><strong>filter (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.filterWith {
  case Train(_, isOnTime) => isOnTime
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>keyingBy</strong></td>
      <td><strong>keyBy (DataStream)</strong></td>
      <td>
{% highlight scala %}
data.keyingBy {
  case (id, _, _) => id
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>mapWith</strong></td>
      <td><strong>map (ConnectedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.mapWith(
  map1 = case (_, value) => value.toString,
  map2 = case (_, _, value, _) => value + 1
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>flatMapWith</strong></td>
      <td><strong>flatMap (ConnectedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.flatMapWith(
  flatMap1 = case (_, json) => parse(json),
  flatMap2 = case (_, _, json, _) => parse(json)
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>keyingBy</strong></td>
      <td><strong>keyBy (ConnectedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.keyingBy(
  key1 = case (_, timestamp) => timestamp,
  key2 = case (id, _, _) => id
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>reduceWith</strong></td>
      <td><strong>reduce (KeyedDataStream, WindowedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.reduceWith {
  case ((_, sum1), (_, sum2) => sum1 + sum2
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>foldWith</strong></td>
      <td><strong>fold (KeyedDataStream, WindowedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.foldWith(User(bought = 0)) {
  case (User(b), (_, items)) => User(b + items.size)
}
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>applyWith</strong></td>
      <td><strong>apply (WindowedDataStream)</strong></td>
      <td>
{% highlight scala %}
data.applyWith(0)(
  foldFunction = case (sum, amount) => sum + amount
  windowFunction = case (k, w, sum) => // [...]
)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>projecting</strong></td>
      <td><strong>apply (JoinedDataStream)</strong></td>
      <td>
{% highlight scala %}
data1.join(data2).
  whereClause(case (pk, _) => pk).
  isEqualTo(case (_, fk) => fk).
  projecting {
    case ((pk, tx), (products, fk)) => tx -> products
  }
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>



获取更多方法的语法信息，请参考
[DataSet]({{ site.baseurl }}/dev/batch/index.html) 和 [DataStream]({{ site.baseurl }}/dev/datastream_api.html) 的 API 帮助文档.

仅仅使用这一个扩展，你可以添加以下导入:

{% highlight scala %}
import org.apache.flink.api.scala.extensions.acceptPartialFunctions
{% endhighlight %}

对数据集扩展导入

{% highlight scala %}
import org.apache.flink.streaming.api.scala.extensions.acceptPartialFunctions
{% endhighlight %}

下面片段展示了如何用数据集 API 使用这些扩展方法的小例子:

{% highlight scala %}
object Main {
  import org.apache.flink.api.scala.extensions._
  case class Point(x: Double, y: Double)
  def main(args: Array[String]): Unit = {
    val env = ExecutionEnvironment.getExecutionEnvironment
    val ds = env.fromElements(Point(1, 2), Point(3, 4), Point(5, 6))
    ds.filterWith {
      case Point(x, _) => x > 1
    }.reduceWith {
      case (Point(x1, y1), (Point(x2, y2))) => Point(x1 + y1, x2 + y2)
    }.mapWith {
      case Point(x, y) => (x, y)
    }.flatMapWith {
      case (x, y) => Seq("x" -> x, "y" -> y)
    }.groupingBy {
      case (id, value) => id
    }
  }
}
{% endhighlight %}
