---
title: Library Methods
nav-parent_id: graphs
nav-pos: 3
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

Gelly 拥有一组图算法来简易分析大规模的图，这些算法至今仍在不断增长。

* This will be replaced by the TOC
{:toc}

Gelly 库的方法能够通简单地过对输入图调用 `run()` 方法来使用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

Graph<Long, Long, NullValue> graph = ...

// 用 30 次迭代运行 Label Propagation 来探测输入图的社区 (communities)
DataSet<Vertex<Long, Long>> verticesWithCommunity = graph.run(new LabelPropagation<Long>(30));

// 打印结果
verticesWithCommunity.print();

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

val graph: Graph[java.lang.Long, java.lang.Long, NullValue] = ...

// 用 30 次迭代运行 Label Propagation 来探测输入图的社区 (communities)
val verticesWithCommunity = graph.run(new LabelPropagation[java.lang.Long, java.lang.Long, NullValue](30))

// 打印结果
verticesWithCommunity.print

{% endhighlight %}
</div>
</div>

## 社区探测 (Community Detection)

#### 概览
在图论中，社区 (communities) 指的是一组对内紧密连接的，但对外与其它组连接稀疏的节点。
该库方法是一个社区探测算法的实现，该算法的具体描述请参阅 [Towards real-time community detection in large networks](http://arxiv.org/pdf/0808.2633.pdf) 这篇论文。

#### 细节
该算法通过使用 [scatter-gather iterations](#scatter-gather-iterations) 进行实现。
最开始，每个顶点被分配一个 `Tuple2` 它包含了其初始值和一个分数， 该分数等于 1.0.
在每一次迭代中，所有顶点将自身的标签和分数发送给它们的邻居。当接受到来自邻居的信息时，顶点选择分数最高的标签并用边值，一个用户指定的跳衰减 (hop attenuation) 参数 `delta`，和超步 (superstep) 数对标签重新算分。
该算法在顶点不再更新它们的值或到达最大迭代次数时收敛。

#### 用法
该算法以一个任何顶点类型的 `Graph` ， `Long` 类型顶点值和 `Double` 类型的边值为输入，返回一个和输入同类型的 `Graph`，
其中顶点值与社区标签 (community labels) 对应， 也就是说如果两个顶点有相同的顶点值，则这两个顶点属于同一个社区。
构造函数接收两个参数：

* `maxIterations`: 要运行的最大迭代数.
* `delta`: 跳衰减参数，默认值为t 0.5.

## Label Propagation

#### 概览
This is an implementation of the well-known Label Propagation algorithm described in [this paper](http://journals.aps.org/pre/abstract/10.1103/PhysRevE.76.036106). The algorithm discovers communities in a graph, by iteratively propagating labels between neighbors. Unlike the [Community Detection library method](#community-detection), this implementation does not use scores associated with the labels.

#### 细节
The algorithm is implemented using [scatter-gather iterations](#scatter-gather-iterations).
Labels are expected to be of type `Comparable` and are initialized using the vertex values of the input `Graph`.
The algorithm iteratively refines discovered communities by propagating labels. In each iteration, a vertex adopts
the label that is most frequent among its neighbors' labels. In case of a tie (i.e. two or more labels appear with the
same frequency), the algorithm picks the greater label. The algorithm converges when no vertex changes its value or
the maximum number of iterations has been reached. Note that different initializations might lead to different results.

#### 用法
The algorithm takes as input a `Graph` with a `Comparable` vertex type, a `Comparable` vertex value type and an arbitrary edge value type.
It returns a `DataSet` of vertices, where the vertex value corresponds to the community in which this vertex belongs after convergence.
The constructor takes one parameter:

* `maxIterations`: the maximum number of iterations to run.

## Connected Components

#### 概览
This is an implementation of the Weakly Connected Components algorithm. Upon convergence, two vertices belong to the
same component, if there is a path from one to the other, without taking edge direction into account.

#### 细节
The algorithm is implemented using [scatter-gather iterations](#scatter-gather-iterations).
This implementation uses a comparable vertex value as initial component identifier (ID). Vertices propagate their
current value in each iteration. Upon receiving component IDs from its neighbors, a vertex adopts a new component ID if
its value is lower than its current component ID. The algorithm converges when vertices no longer update their component
ID value or when the maximum number of iterations has been reached.

#### 用法
The result is a `DataSet` of vertices, where the vertex value corresponds to the assigned component.
The constructor takes one parameter:

* `maxIterations`: the maximum number of iterations to run.

## GSA Connected Components

#### 概览
This is an implementation of the Weakly Connected Components algorithm. Upon convergence, two vertices belong to the
same component, if there is a path from one to the other, without taking edge direction into account.

#### 细节
The algorithm is implemented using [gather-sum-apply iterations](#gather-sum-apply-iterations).
This implementation uses a comparable vertex value as initial component identifier (ID). In the gather phase, each
vertex collects the vertex value of their adjacent vertices. In the sum phase, the minimum among those values is
selected. In the apply phase, the algorithm sets the minimum value as the new vertex value if it is smaller than
the current value. The algorithm converges when vertices no longer update their component ID value or when the
maximum number of iterations has been reached.

#### 用法
The result is a `DataSet` of vertices, where the vertex value corresponds to the assigned component.
The constructor takes one parameter:

* `maxIterations`: the maximum number of iterations to run.

## Single Source Shortest Paths

#### 概览
An implementation of the Single-Source-Shortest-Paths algorithm for weighted graphs. Given a source vertex, the algorithm computes the shortest paths from this source to all other nodes in the graph.

#### 细节
The algorithm is implemented using [scatter-gather iterations](#scatter-gather-iterations).
In each iteration, a vertex sends to its neighbors a message containing the sum its current distance and the edge weight connecting this vertex with the neighbor. Upon receiving candidate distance messages, a vertex calculates the minimum distance and, if a shorter path has been discovered, it updates its value. If a vertex does not change its value during a superstep, then it does not produce messages for its neighbors for the next superstep. The computation terminates after the specified maximum number of supersteps or when there are no value updates.

#### 用法
The algorithm takes as input a `Graph` with any vertex type and `Double` edge values. The vertex values can be any type and are not used by this algorithm. The vertex type must implement `equals()`.
The output is a `DataSet` of vertices where the vertex values correspond to the minimum distances from the given source vertex.
The constructor takes two parameters:

* `srcVertexId` The vertex ID of the source vertex.
* `maxIterations`: the maximum number of iterations to run.

## GSA Single Source Shortest Paths

The algorithm is implemented using [gather-sum-apply iterations](#gather-sum-apply-iterations).

See the [Single Source Shortest Paths](#single-source-shortest-paths) library method for implementation details and usage information.

## Triangle Enumerator

#### 概览
This library method enumerates unique triangles present in the input graph. A triangle consists of three edges that connect three vertices with each other.
This implementation ignores edge directions.

#### 细节
The basic triangle enumeration algorithm groups all edges that share a common vertex and builds triads, i.e., triples of vertices
that are connected by two edges. Then, all triads are filtered for which no third edge exists that closes the triangle.
For a group of <i>n</i> edges that share a common vertex, the number of built triads is quadratic <i>((n*(n-1))/2)</i>.
Therefore, an optimization of the algorithm is to group edges on the vertex with the smaller output degree to reduce the number of triads.
This implementation extends the basic algorithm by computing output degrees of edge vertices and grouping on edges on the vertex with the smaller degree.

#### 用法
The algorithm takes a directed graph as input and outputs a `DataSet` of `Tuple3`. The Vertex ID type has to be `Comparable`.
Each `Tuple3` corresponds to a triangle, with the fields containing the IDs of the vertices forming the triangle.

## Summarization

#### 概览
The summarization algorithm computes a condensed version of the input graph by grouping vertices and edges based on
their values. In doing so, the algorithm helps to uncover insights about patterns and distributions in the graph.
One possible use case is the visualization of communities where the whole graph is too large and needs to be summarized
based on the community identifier stored at a vertex.

#### 细节
In the resulting graph, each vertex represents a group of vertices that share the same value. An edge, that connects a
vertex with itself, represents all edges with the same edge value that connect vertices from the same vertex group. An
edge between different vertices in the output graph represents all edges with the same edge value between members of
different vertex groups in the input graph.

The algorithm is implemented using Flink data operators. First, vertices are grouped by their value and a representative
is chosen from each group. For any edge, the source and target vertex identifiers are replaced with the corresponding
representative and grouped by source, target and edge value. Output vertices and edges are created from their
corresponding groupings.

#### 用法
The algorithm takes a directed, vertex (and possibly edge) attributed graph as input and outputs a new graph where each
vertex represents a group of vertices and each edge represents a group of edges from the input graph. Furthermore, each
vertex and edge in the output graph stores the common group value and the number of represented elements.

## Clustering

### Average Clustering Coefficient

#### 概览
The average clustering coefficient measures the mean connectedness of a graph. Scores range from 0.0 (no edges between
neighbors) to 1.0 (complete graph).

#### 细节
See the [Local Clustering Coefficient](#local-clustering-coefficient) library method for a detailed explanation of
clustering coefficient. The Average Clustering Coefficient is the average of the Local Clustering Coefficient scores
over all vertices with at least two neighbors. Each vertex, independent of degree, has equal weight for this score.

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an `AnalyticResult`
containing the total number of vertices and average clustering coefficient of the graph. The graph ID type must be
`Comparable` and `Copyable`.

* `setLittleParallelism`: override the parallelism of operators processing small amounts of data

### Global Clustering Coefficient

#### 概览
The global clustering coefficient measures the connectedness of a graph. Scores range from 0.0 (no edges between
neighbors) to 1.0 (complete graph).

#### 细节
See the [Local Clustering Coefficient](#local-clustering-coefficient) library method for a detailed explanation of
clustering coefficient. The Global Clustering Coefficient is the ratio of connected neighbors over the entire graph.
Vertices with higher degrees have greater weight for this score because the count of neighbor pairs is quadratic in
degree.

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an `AnalyticResult`
containing the total number of triplets and triangles in the graph. The result class provides a method to compute the
global clustering coefficient score. The graph ID type must be `Comparable` and `Copyable`.

* `setLittleParallelism`: override the parallelism of operators processing small amounts of data

### Local Clustering Coefficient

#### 概览
The local clustering coefficient measures the connectedness of each vertex's neighborhood. Scores range from 0.0 (no
edges between neighbors) to 1.0 (neighborhood is a clique).

#### 细节
An edge between neighbors of a vertex is a triangle. Counting edges between neighbors is equivalent to counting the
number of triangles which include the vertex. The clustering coefficient score is the number of edges between neighbors
divided by the number of potential edges between neighbors.

See the [Triangle Listing](#triangle-listing) library method for a detailed explanation of triangle enumeration.

#### 用法
Directed and undirected variants are provided. The algorithms take a simple graph as input and output a `DataSet` of
`UnaryResult` containing the vertex ID, vertex degree, and number of triangles containing the vertex. The result class
provides a method to compute the local clustering coefficient score. The graph ID type must be `Comparable` and
`Copyable`.

* `setIncludeZeroDegreeVertices`: include results for vertices with a degree of zero
* `setLittleParallelism`: override the parallelism of operators processing small amounts of data

### Triadic Census

#### 概览
A triad is formed by any three vertices in a graph. Each triad contains three pairs of vertices which may be connected
or unconnected. The [Triadic Census](http://vlado.fmf.uni-lj.si/pub/networks/doc/triads/triads.pdf) counts the
occurrences of each type of triad with the graph.

#### 细节
This analytic counts the four undirected triad types (formed with 0, 1, 2, or 3 connecting edges) or 16 directed triad
types by counting the triangles from [Triangle Listing](#triangle-listing) and running [Vertex Metrics](#vertex-metrics)
to obtain the number of triplets and edges. Triangle counts are then deducted from triplet counts, and triangle and
triplet counts are removed from edge counts.

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an
`AnalyticResult` with accessor methods for querying the count of each triad type. The graph ID type must be
`Comparable` and `Copyable`.

* `setLittleParallelism`: override the parallelism of operators processing small amounts of data

### Triangle Listing

#### 概览
Enumerates all triangles in the graph. A triangle is composed of three edges connecting three vertices into cliques of
size 3.

#### 细节
Triangles are listed by joining open triplets (two edges with a common neighbor) against edges on the triplet endpoints.
This implementation uses optimizations from
[Schank's algorithm](http://i11www.iti.uni-karlsruhe.de/extra/publications/sw-fclt-05_t.pdf) to improve performance with
high-degree vertices. Triplets are generated from the lowest degree vertex since each triangle need only be listed once.
This greatly reduces the number of generated triplets which is quadratic in vertex degree.

#### 用法
Directed and undirected variants are provided. The algorithms take a simple graph as input and output a `DataSet` of
`TertiaryResult` containing the three triangle vertices and, for the directed algorithm, a bitmask marking each of the
six potential edges connecting the three vertices. The graph ID type must be `Comparable` and `Copyable`.

* `setLittleParallelism`: override the parallelism of operators processing small amounts of data
* `setSortTriangleVertices`: normalize the triangle listing such that for each result (K0, K1, K2) the vertex IDs are sorted K0 < K1 < K2

## Link Analysis

### Hyperlink-Induced Topic Search

#### 概览
[Hyperlink-Induced Topic Search](http://www.cs.cornell.edu/home/kleinber/auth.pdf) (HITS, or "Hubs and Authorities")
computes two interdependent scores for every vertex in a directed graph. Good hubs are those which point to many
good authorities and good authorities are those pointed to by many good hubs.

#### 细节
Every vertex is assigned the same initial hub and authority scores. The algorithm then iteratively updates the scores
until termination. During each iteration new hub scores are computed from the authority scores, then new authority
scores are computed from the new hub scores. The scores are then normalized and optionally tested for convergence.
HITS is similar to [PageRank](#pagerank) but vertex scores are emitted in full to each neighbor whereas in PageRank
the vertex score is first divided by the number of neighbors.

#### 用法
The algorithm takes a simple directed graph as input and outputs a `DataSet` of `UnaryResult` containing the vertex ID,
hub score, and authority score. Termination is configured by the number of iterations and/or a convergence threshold on
the iteration sum of the change in scores over all vertices.

* `setParallelism`: override the operator parallelism

### PageRank

#### 概览
[PageRank](https://en.wikipedia.org/wiki/PageRank) is an algorithm that was first used to rank web search engine
results. Today, the algorithm and many variations are used in various graph application domains. The idea of PageRank is
that important or relevant vertices tend to link to other important vertices.

#### 细节
The algorithm operates in iterations, where pages distribute their scores to their neighbors (pages they have links to)
and subsequently update their scores based on the sum of values they receive. In order to consider the importance of a
link from one page to another, scores are divided by the total number of out-links of the source page. Thus, a page with
10 links will distribute 1/10 of its score to each neighbor, while a page with 100 links will distribute 1/100 of its
score to each neighboring page.

#### 用法
The algorithm takes a directed graph as input and outputs a `DataSet` where each `Result` contains the vertex ID and
PageRank score. Termination is configured with a maximum number of iterations and/or a convergence threshold
on the sum of the change in score for each vertex between iterations.

* `setParallelism`: override the operator parallelism

## Metric

### Vertex Metrics

#### 概览
This graph analytic computes the following statistics for both directed and undirected graphs:
- number of vertices
- number of edges
- average degree
- number of triplets
- maximum degree
- maximum number of triplets

The following statistics are additionally computed for directed graphs:
- number of unidirectional edges
- number of bidirectional edges
- maximum out degree
- maximum in degree

#### 细节
The statistics are computed over vertex degrees generated from `degree.annotate.directed.VertexDegrees` or
`degree.annotate.undirected.VertexDegree`.

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an `AnalyticResult`
with accessor methods for the computed statistics. The graph ID type must be `Comparable`.

* `setIncludeZeroDegreeVertices`: include results for vertices with a degree of zero
* `setParallelism`: override the operator parallelism
* `setReduceOnTargetId` (undirected only): the degree can be counted from either the edge source or target IDs. By default the source IDs are counted. Reducing on target IDs may optimize the algorithm if the input edge list is sorted by target ID

### Edge Metrics

#### 概览
This graph analytic computes the following statistics:
- number of triangle triplets
- number of rectangle triplets
- maximum number of triangle triplets
- maximum number of rectangle triplets

#### 细节
The statistics are computed over edge degrees generated from `degree.annotate.directed.EdgeDegreesPair` or
`degree.annotate.undirected.EdgeDegreePair` and grouped by vertex.

#### 用法
Directed and undirected variants are provided. The analytics take a simple graph as input and output an `AnalyticResult`
with accessor methods for the computed statistics. The graph ID type must be `Comparable`.

* `setParallelism`: override the operator parallelism
* `setReduceOnTargetId` (undirected only): the degree can be counted from either the edge source or target IDs. By default the source IDs are counted. Reducing on target IDs may optimize the algorithm if the input edge list is sorted by target ID

## Similarity

### Adamic-Adar

#### 概览
Adamic-Adar measures the similarity between pairs of vertices as the sum of the inverse logarithm of degree over shared
neighbors. Scores are non-negative and unbounded. A vertex with higher degree has greater overall influence but is less
influential to each pair of neighbors.

#### 细节
The algorithm first annotates each vertex with the inverse of the logarithm of the vertex degree then joins this score
onto edges by source vertex. Grouping on the source vertex, each pair of neighbors is emitted with the vertex score.
Grouping on vertex pairs, the Adamic-Adar score is summed.

See the [Jaccard Index](#jaccard-index) library method for a similar algorithm.

#### 用法
The algorithm takes a simple undirected graph as input and outputs a `DataSet` of `BinaryResult` containing two vertex
IDs and the Adamic-Adar similarity score. The graph ID type must be `Copyable`.

* `setLittleParallelism`: override the parallelism of operators processing small amounts of data
* `setMinimumRatio`: filter out Adamic-Adar scores less than the given ratio times the average score
* `setMinimumScore`: filter out Adamic-Adar scores less than the given minimum

### Jaccard Index

#### 概览
The Jaccard Index measures the similarity between vertex neighborhoods and is computed as the number of shared neighbors
divided by the number of distinct neighbors. Scores range from 0.0 (no shared neighbors) to 1.0 (all neighbors are
shared).

#### 细节
Counting shared neighbors for pairs of vertices is equivalent to counting connecting paths of length two. The number of
distinct neighbors is computed by storing the sum of degrees of the vertex pair and subtracting the count of shared
neighbors, which are double-counted in the sum of degrees.

The algorithm first annotates each edge with the target vertex's degree. Grouping on the source vertex, each pair of
neighbors is emitted with the degree sum. Grouping on vertex pairs, the shared neighbors are counted.

#### 用法
The algorithm takes a simple undirected graph as input and outputs a `DataSet` of tuples containing two vertex IDs,
the number of shared neighbors, and the number of distinct neighbors. The result class provides a method to compute the
Jaccard Index score. The graph ID type must be `Copyable`.

* `setLittleParallelism`: override the parallelism of operators processing small amounts of data
* `setMaximumScore`: filter out Jaccard Index scores greater than or equal to the given maximum fraction
* `setMinimumScore`: filter out Jaccard Index scores less than the given minimum fraction

{% top %}
