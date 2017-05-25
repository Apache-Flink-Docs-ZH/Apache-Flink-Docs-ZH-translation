---
title: Graph Generators
nav-parent_id: graphs
nav-pos: 5
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

* This will be replaced by the TOC
{:toc}

Gelly 提供了一组可扩展的图生成器。每个生成器都是：

* 并行的, 用于创建大型数据集。
* 自由扩展的, 用于生成并行度无关的同样的图。
* 简洁的，使用了尽可能少的操作。

图生成器使用Builder模式进行配置，可以通过调用`setParallelism(parallelism)`设置并行度。减少
并行度可以降低内存和网络缓冲区的使用。

特定的图配置必须首先被调用，该配置对所有的图生成器都是通用的，最后才会调用`generate()`。
接下来的例子使用两个维度配置了网格图，配置了并行度并生成了图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

boolean wrapEndpoints = false;

int parallelism = 4;

Graph<LongValue,NullValue,NullValue> graph = new GridGraph(env)
    .addDimension(2, wrapEndpoints)
    .addDimension(4, wrapEndpoints)
    .setParallelism(parallelism)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.GridGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

wrapEndpoints = false

val parallelism = 4

val graph = new GridGraph(env.getJavaEnv).addDimension(2, wrapEndpoints).addDimension(4, wrapEndpoints).setParallelism(parallelism).generate()
{% endhighlight %}
</div>
</div>

## 完全图

连接所有不同顶点对的无向图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5;

Graph<LongValue,NullValue,NullValue> graph = new CompleteGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.CompleteGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new CompleteGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="540"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="270" y1="40" x2="489" y2="199" />
    <line x1="270" y1="40" x2="405" y2="456" />
    <line x1="270" y1="40" x2="135" y2="456" />
    <line x1="270" y1="40" x2="51" y2="199" />

    <line x1="489" y1="199" x2="405" y2="456" />
    <line x1="489" y1="199" x2="135" y2="456" />
    <line x1="489" y1="199" x2="51" y2="199" />

    <line x1="405" y1="456" x2="135" y2="456" />
    <line x1="405" y1="456" x2="51" y2="199" />

    <line x1="135" y1="456" x2="51" y2="199" />

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">0</text>

    <circle cx="489" cy="199" r="20" />
    <text x="489" y="199">1</text>

    <circle cx="405" cy="456" r="20" />
    <text x="405" y="456">2</text>

    <circle cx="135" cy="456" r="20" />
    <text x="135" y="456">3</text>

    <circle cx="51" cy="199" r="20" />
    <text x="51" y="199">4</text>
</svg>

## 环图

所有的边形成一个环的无向图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5;

Graph<LongValue,NullValue,NullValue> graph = new CycleGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.CycleGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new CycleGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="540"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="270" y1="40" x2="489" y2="199" />
    <line x1="489" y1="199" x2="405" y2="456" />
    <line x1="405" y1="456" x2="135" y2="456" />
    <line x1="135" y1="456" x2="51" y2="199" />
    <line x1="51" y1="199" x2="270" y2="40" />

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">0</text>

    <circle cx="489" cy="199" r="20" />
    <text x="489" y="199">1</text>

    <circle cx="405" cy="456" r="20" />
    <text x="405" y="456">2</text>

    <circle cx="135" cy="456" r="20" />
    <text x="135" y="456">3</text>

    <circle cx="51" cy="199" r="20" />
    <text x="51" y="199">4</text>
</svg>

## 空图

不存在边的图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5;

Graph<LongValue,NullValue,NullValue> graph = new EmptyGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.EmptyGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new EmptyGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="80"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">0</text>

    <circle cx="150" cy="40" r="20" />
    <text x="150" y="40">1</text>

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">2</text>

    <circle cx="390" cy="40" r="20" />
    <text x="390" y="40">3</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">4</text>
</svg>

## 网格图

一种点在一到多个维度正常平铺的无向图。每个维度都是独立配置的。当维度大小多于3时，每个维度的端点
可以通过设置`wrapEndpoints`连接起来，那么下边例子的`addDimension(4, true)`将会连接`0`和`3`
以及`4`和`7`。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

boolean wrapEndpoints = false;

Graph<LongValue,NullValue,NullValue> graph = new GridGraph(env)
    .addDimension(2, wrapEndpoints)
    .addDimension(4, wrapEndpoints)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.GridGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val wrapEndpoints = false

val graph = new GridGraph(env.getJavaEnv).addDimension(2, wrapEndpoints).addDimension(4, wrapEndpoints).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="200"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="30" y1="40" x2="510" y2="40" />
    <line x1="30" y1="160" x2="510" y2="160" />

    <line x1="30" y1="40" x2="30" y2="160" />
    <line x1="190" y1="40" x2="190" y2="160" />
    <line x1="350" y1="40" x2="350" y2="160" />
    <line x1="510" y1="40" x2="510" y2="160" />

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">0</text>

    <circle cx="190" cy="40" r="20" />
    <text x="190" y="40">1</text>

    <circle cx="350" cy="40" r="20" />
    <text x="350" y="40">2</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">3</text>

    <circle cx="30" cy="160" r="20" />
    <text x="30" y="160">4</text>

    <circle cx="190" cy="160" r="20" />
    <text x="190" y="160">5</text>

    <circle cx="350" cy="160" r="20" />
    <text x="350" y="160">6</text>

    <circle cx="510" cy="160" r="20" />
    <text x="510" y="160">7</text>
</svg>

## 超立方体图

所有的边形成N维超立方体的无向图。超立方体内的每个顶点和同维度的其他顶点连接。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long dimensions = 3;

Graph<LongValue,NullValue,NullValue> graph = new HypercubeGraph(env, dimensions)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.HypercubeGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val dimensions = 3

val graph = new HypercubeGraph(env.getJavaEnv, dimensions).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="320"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="190" y1="120" x2="350" y2="120" />
    <line x1="190" y1="200" x2="350" y2="200" />
    <line x1="190" y1="120" x2="190" y2="200" />
    <line x1="350" y1="120" x2="350" y2="200" />

    <line x1="30" y1="40" x2="510" y2="40" />
    <line x1="30" y1="280" x2="510" y2="280" />
    <line x1="30" y1="40" x2="30" y2="280" />
    <line x1="510" y1="40" x2="510" y2="280" />

    <line x1="190" y1="120" x2="30" y2="40" />
    <line x1="350" y1="120" x2="510" y2="40" />
    <line x1="190" y1="200" x2="30" y2="280" />
    <line x1="350" y1="200" x2="510" y2="280" />

    <circle cx="190" cy="120" r="20" />
    <text x="190" y="120">0</text>

    <circle cx="350" cy="120" r="20" />
    <text x="350" y="120">1</text>

    <circle cx="190" cy="200" r="20" />
    <text x="190" y="200">2</text>

    <circle cx="350" cy="200" r="20" />
    <text x="350" y="200">3</text>

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">4</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">5</text>

    <circle cx="30" cy="280" r="20" />
    <text x="30" y="280">6</text>

    <circle cx="510" cy="280" r="20" />
    <text x="510" y="280">7</text>
</svg>

## 路径图

所有的边形成了一条路径的无向图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 5

Graph<LongValue,NullValue,NullValue> graph = new PathGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.PathGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 5

val graph = new PathGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="80"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="30" y1="40" x2="510" y2="40" />

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">0</text>

    <circle cx="150" cy="40" r="20" />
    <text x="150" y="40">1</text>

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">2</text>

    <circle cx="390" cy="40" r="20" />
    <text x="390" y="40">3</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">4</text>
</svg>

## RMat图

使用[Recursive Matrix (R-Mat)](http://www.cs.cmu.edu/~christos/PUBLICATIONS/siam04.pdf)模型
生成的有向或者无向幂图。

RMat是一个使用实现`RandomGenerableFactory`接口的随机源配置的随机生成器，`JDKRandomGeneratorFactory`
和`MersenneTwisterFactory`实现了该接口。它产生了一个用于生成边的随机种子的随机初始序列。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

RandomGenerableFactory<JDKRandomGenerator> rnd = new JDKRandomGeneratorFactory();

int vertexCount = 1 << scale;
int edgeCount = edgeFactor * vertexCount;

Graph<LongValue,NullValue,NullValue> graph = new RMatGraph<>(env, rnd, vertexCount, edgeCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.RMatGraph

val env = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 1 << scale
val edgeCount = edgeFactor * vertexCount

val graph = new RMatGraph(env.getJavaEnv, rnd, vertexCount, edgeCount).generate()
{% endhighlight %}
</div>
</div>

The default RMat contants can be overridden as shown in the following example.
The contants define the interdependence of bits from each generated edge's source
and target labels. The RMat noise can be enabled and progressively perturbs the
contants while generating each edge.

The RMat generator can be configured to produce a simple graph by removing self-loops
and duplicate edges. Symmetrization is performed either by a "clip-and-flip" throwing away
the half matrix above the diagonal or a full "flip" preserving and mirroring all edges.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

RandomGenerableFactory<JDKRandomGenerator> rnd = new JDKRandomGeneratorFactory();

int vertexCount = 1 << scale;
int edgeCount = edgeFactor * vertexCount;

boolean clipAndFlip = false;

Graph<LongValue,NullValue,NullValue> graph = new RMatGraph<>(env, rnd, vertexCount, edgeCount)
    .setConstants(0.57f, 0.19f, 0.19f)
    .setNoise(true, 0.10f)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.RMatGraph

val env = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 1 << scale
val edgeCount = edgeFactor * vertexCount

clipAndFlip = false

val graph = new RMatGraph(env.getJavaEnv, rnd, vertexCount, edgeCount).setConstants(0.57f, 0.19f, 0.19f).setNoise(true, 0.10f).generate()
{% endhighlight %}
</div>
</div>

## 单边图

包含独立的双路径的无向图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexPairCount = 4

// note: configured with the number of vertex pairs
Graph<LongValue,NullValue,NullValue> graph = new SingletonEdgeGraph(env, vertexPairCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.SingletonEdgeGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexPairCount = 4

// note: configured with the number of vertex pairs
val graph = new SingletonEdgeGraph(env.getJavaEnv, vertexPairCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="200"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="30" y1="40" x2="190" y2="40" />
    <line x1="350" y1="40" x2="510" y2="40" />
    <line x1="30" y1="160" x2="190" y2="160" />
    <line x1="350" y1="160" x2="510" y2="160" />

    <circle cx="30" cy="40" r="20" />
    <text x="30" y="40">0</text>

    <circle cx="190" cy="40" r="20" />
    <text x="190" y="40">1</text>

    <circle cx="350" cy="40" r="20" />
    <text x="350" y="40">2</text>

    <circle cx="510" cy="40" r="20" />
    <text x="510" y="40">3</text>

    <circle cx="30" cy="160" r="20" />
    <text x="30" y="160">4</text>

    <circle cx="190" cy="160" r="20" />
    <text x="190" y="160">5</text>

    <circle cx="350" cy="160" r="20" />
    <text x="350" y="160">6</text>

    <circle cx="510" cy="160" r="20" />
    <text x="510" y="160">7</text>
</svg>

## 星图

包含一个连接到所有其他叶子顶点的中心顶点的无向图。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

long vertexCount = 6;

Graph<LongValue,NullValue,NullValue> graph = new StarGraph(env, vertexCount)
    .generate();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.graph.generator.StarGraph

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

val vertexCount = 6

val graph = new StarGraph(env.getJavaEnv, vertexCount).generate()
{% endhighlight %}
</div>
</div>

<svg class="graph" width="540" height="540"
    xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <line x1="270" y1="270" x2="270" y2="40" />
    <line x1="270" y1="270" x2="489" y2="199" />
    <line x1="270" y1="270" x2="405" y2="456" />
    <line x1="270" y1="270" x2="135" y2="456" />
    <line x1="270" y1="270" x2="51" y2="199" />

    <circle cx="270" cy="270" r="20" />
    <text x="270" y="270">0</text>

    <circle cx="270" cy="40" r="20" />
    <text x="270" y="40">1</text>

    <circle cx="489" cy="199" r="20" />
    <text x="489" y="199">2</text>

    <circle cx="405" cy="456" r="20" />
    <text x="405" y="456">3</text>

    <circle cx="135" cy="456" r="20" />
    <text x="135" y="456">4</text>

    <circle cx="51" cy="199" r="20" />
    <text x="51" y="199">5</text>
</svg>

{% top %}
