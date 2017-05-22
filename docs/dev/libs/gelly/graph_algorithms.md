---
title: Graph Algorithms
nav-parent_id: graphs
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

在 Gelly 中作为图算法，`Graph` API 和 顶层算法集成的逻辑模块都在 `org.apache.flink.graph.asm` 中。这些算法可用通过配置参数进行优化和调整，并且当用一组相似的配置对相同的输入进行处理时，提供隐式的运行时复用。.

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">算法</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td>degree.annotate.directed.<br/><strong>VertexInDegree</strong></td>
      <td>
        <p>用入边 (in-degree) 标注一个<a href="#graph-representation">有向图</a>的点.</p>
{% highlight java %}
DataSet<Vertex<K, LongValue>> inDegree = graph
  .run(new VertexInDegree()
    .setIncludeZeroDegreeVertices(true));
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setIncludeZeroDegreeVertices</strong>: 默认情况下为了自由度的计算，只有边集 (edge set) 需要被处理；当该参数被设置时，对点集 (vertex set) 会进行一个额外的 join 操作来输出入边数 (in-degree) 为 0 的点</p></li>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度 </p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.directed.<br/><strong>VertexOutDegree</strong></td>
      <td>
        <p>用出边 (out-degree) 标注一个<a href="#graph-representation">有向图</a>的点.</p>
{% highlight java %}
DataSet<Vertex<K, LongValue>> outDegree = graph
  .run(new VertexOutDegree()
    .setIncludeZeroDegreeVertices(true));
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setIncludeZeroDegreeVertices</strong>: 默认情况下为了自由度的计算，只有边集 (edge set) 需要被处理；当该参数被设置时，对点集 (vertex set) 会进行一个额外的 join 操作来输出出边数 (out-degree) 为 0 的点</p></li>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.directed.<br/><strong>VertexDegrees</strong></td>
      <td>
        <p>用自由度（degree）, 出边（out-degree）, 和入边（in-degree）标注一个<a href="#graph-representation">有向图</a>的点.</p>
{% highlight java %}
DataSet<Vertex<K, Tuple2<LongValue, LongValue>>> degrees = graph
  .run(new VertexDegrees()
    .setIncludeZeroDegreeVertices(true));
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setIncludeZeroDegreeVertices</strong>: 默认情况下为了自由度的计算，只有边集 (edge set) 需要被处理；当该参数被设置时，对点集 (vertex set) 会进行一个额外的 join 操作来输出出边数 (out-degree) 和入边数 (in-degree) 为 0 的点</p></li>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.directed.<br/><strong>EdgeSourceDegrees</strong></td>
      <td>
        <p>用源点的自由度（degree），出边（out-degree）和入边（in-degree）标注一个<a href="#graph-representation">有向图</a>的边.</p>
{% highlight java %}
DataSet<Edge<K, Tuple2<EV, Degrees>>> sourceDegrees = graph
  .run(new EdgeSourceDegrees());
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度/p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.directed.<br/><strong>EdgeTargetDegrees</strong></td>
      <td>
        <p>用目标点的自由度（degree），出边（out-degree）和入边（in-degree）标注一个<a href="#graph-representation">有向图</a>的边.</p>
{% highlight java %}
DataSet<Edge<K, Tuple2<EV, Degrees>>> targetDegrees = graph
  .run(new EdgeTargetDegrees();
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.directed.<br/><strong>EdgeDegreesPair</strong></td>
      <td>
        <p>用源点目标点的自由度（degree），出边（out-degree）和入边（in-degree）标注一个<a href="#graph-representation">有向图</a>的边.</p>
{% highlight java %}
DataSet<Edge<K, Tuple2<EV, Degrees>>> degrees = graph
  .run(new EdgeDegreesPair());
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.undirected.<br/><strong>VertexDegree</strong></td>
      <td>
        <p>用自由度（degree）标注一个<a href="#graph-representation">无向图</a>的点.</p>
{% highlight java %}
DataSet<Vertex<K, LongValue>> degree = graph
  .run(new VertexDegree()
    .setIncludeZeroDegreeVertices(true)
    .setReduceOnTargetId(true));
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setIncludeZeroDegreeVertices</strong>: 默认情况下为了自由度的计算，只有边集 (edge set) 需要被处理；当该参数被设置时，对点集 (vertex set) 会进行一个额外的 join 操作来输出自由度 (degree) 为 0 的点</p></li>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
          <li><p><strong>setReduceOnTargetId</strong>: 自由度能够用边的源点和终点计算. 默认情况下用源点计算. 如果用目标点对输入边列 (edge list) 排序，对终点的归约可能优化该算法.</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.undirected.<br/><strong>EdgeSourceDegree</strong></td>
      <td>
        <p>用源点的自由度（degree）标注一个<a href="#graph-representation">无向图</a>的边.</p>
{% highlight java %}
DataSet<Edge<K, Tuple2<EV, LongValue>>> sourceDegree = graph
  .run(new EdgeSourceDegree()
    .setReduceOnTargetId(true));
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
          <li><p><strong>setReduceOnTargetId</strong>: 自由度能够用边的源点和终点计算. 默认情况下用源点计算. 如果用目标点对输入边列 (edge list) 排序，对终点的归约可能优化该算法.</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.undirected.<br/><strong>EdgeTargetDegree</strong></td>
      <td>
        <p>用目标点的自由度（degree）标注一个<a href="#graph-representation">无向图</a>的边.</p>
{% highlight java %}
DataSet<Edge<K, Tuple2<EV, LongValue>>> targetDegree = graph
  .run(new EdgeTargetDegree()
    .setReduceOnSourceId(true));
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
          <li><p><strong>setReduceOnSourceId</strong>: 自由度能够用边的源点和终点计算. 默认情况下用源点计算. 如果用目标点对输入边列 (edge list) 排序，对终点的归约可能优化该算法.</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.annotate.undirected.<br/><strong>EdgeDegreePair</strong></td>
      <td>
        <p>用源点和目标点的自由度（degree）标注一个<a href="#graph-representation">无向图</a>的边.</p>
{% highlight java %}
DataSet<Edge<K, Tuple3<EV, LongValue, LongValue>>> pairDegree = graph
  .run(new EdgeDegreePair()
    .setReduceOnTargetId(true));
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
          <li><p><strong>setReduceOnTargetId</strong>: 自由度能够用边的源点和终点计算. 默认情况下用源点计算. 如果用目标点对输入边列 (edge list) 排序，对终点的归约可能优化该算法.</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>degree.filter.undirected.<br/><strong>MaximumDegree</strong></td>
      <td>
        <p>用最大自由度过滤一个<a href="#graph-representation">无向图</a>.</p>
{% highlight java %}
Graph<K, VV, EV> filteredGraph = graph
  .run(new MaximumDegree(5000)
    .setBroadcastHighDegreeVertices(true)
    .setReduceOnTargetId(true));
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setBroadcastHighDegreeVertices</strong>: 当移除少量高自由度 (high-degree) 的点时，用一个广播哈希 (broadcast-hash) 合并 (join) 高自由度的点来减少数据洗牌 (shuffle).</p></li>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
          <li><p><strong>setReduceOnTargetId</strong>: 自由度能够用边的源点和终点计算. 默认情况下用源点计算. 如果用目标点对输入边列 (edge list) 排序，对终点的归约可能优化该算法.</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>simple.directed.<br/><strong>Simplify</strong></td>
      <td>
        <p>移除一个<a href="#graph-representation">有向图</a>的自环 (self-loops) 和相同的边.</p>
{% highlight java %}
graph.run(new Simplify());
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>simple.undirected.<br/><strong>Simplify</strong></td>
      <td>
        <p>从一个<a href="#graph-representation">无向图</a>中添加对称边并移除自环 (self-loops).</p>
{% highlight java %}
graph.run(new Simplify());
{% endhighlight %}
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>translate.<br/><strong>TranslateGraphIds</strong></td>
      <td>
        <p>用给定的 <code>TranslateFunction</code> 转换 (translate) 点和边的 ID.</p>
{% highlight java %}
graph.run(new TranslateGraphIds(new LongValueToStringValue()));
{% endhighlight %}
        <p>必要配置:</p>
        <ul>
          <li><p><strong>translator</strong>: 实现类型或值转换</p></li>
        </ul>
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>translate.<br/><strong>TranslateVertexValues</strong></td>
      <td>
        <p>用给定的 <code>TranslateFunction</code> 转换 (translate) 点的值.</p>
{% highlight java %}
graph.run(new TranslateVertexValues(new LongValueAddOffset(vertexCount)));
{% endhighlight %}
        <p>必要配置:</p>
        <ul>
          <li><p><strong>translator</strong>: 实现类型或值转换</p></li>
        </ul>
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>

    <tr>
      <td>translate.<br/><strong>TranslateEdgeValues</strong></td>
      <td>
        <p>用给定的 <code>TranslateFunction</code> 转换 (translate) 边的值.</p>
{% highlight java %}
graph.run(new TranslateEdgeValues(new Nullify()));
{% endhighlight %}
        <p>必要配置:</p>
        <ul>
          <li><p><strong>translator</strong>: 实现类型或值转换</p></li>
        </ul>
        <p>可选配置:</p>
        <ul>
          <li><p><strong>setParallelism</strong>: 指定算子的并行度</p></li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

{% top %}
