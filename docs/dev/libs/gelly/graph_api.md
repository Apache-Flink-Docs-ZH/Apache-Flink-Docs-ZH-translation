---
title: Graph API
nav-parent_id: graphs
nav-pos: 1
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

Graph Representation图的表示
-----------

In Gelly, a `Graph` is represented by a `DataSet` of vertices and a `DataSet` of edges.  
在Gelly中， `图(Graph)`由顶点(vertex)的`DataSet` 和边(edge)的`DataSet`表示。  

The `Graph` nodes are represented by the `Vertex` type. A `Vertex` is defined by a unique ID and a value. `Vertex` IDs should implement the `Comparable` interface. Vertices without value can be represented by setting the value type to `NullValue`.  
`图`的顶点由`Vertex`类表示。`Vertex`由一个唯一ID 和一个value 定义。`Vertex`ID 应该实现`Comparable`接口。要表示没有value的顶点，可以将value的类型设为`NullType`。  


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// create a new vertex with a Long ID and a String value
// 用Long 类型的ID 和String 类型的 value 新建一个顶点
Vertex<Long, String> v = new Vertex<Long, String>(1L, "foo");

// create a new vertex with a Long ID and no value
// 用一个Long 类型的ID 和空value 新建一个顶点
Vertex<Long, NullValue> v = new Vertex<Long, NullValue>(1L, NullValue.getInstance());
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// create a new vertex with a Long ID and a String value
// 用Long 类型的ID 和String 类型的 value 新建一个顶点
val v = new Vertex(1L, "foo")

// create a new vertex with a Long ID and no value
// 用一个Long 类型的ID 和空value 新建一个顶点
val v = new Vertex(1L, NullValue.getInstance())
{% endhighlight %}
</div>
</div>

The graph edges are represented by the `Edge` type. An `Edge` is defined by a source ID (the ID of the source `Vertex`), a target ID (the ID of the target `Vertex`) and an optional value. The source and target IDs should be of the same type as the `Vertex` IDs. Edges with no value have a `NullValue` value type.  
图的边用`Edge`类表示。`Edge`由一个源ID (即源`Vertex`的ID)，一个目的ID (即目的`Vertex`的ID)，一个可选的value 定义。源ID 和目的ID 应该与`Vertex`的ID 属于相同的类。没有值的边，它的value 类型为`NullValue`。  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Edge<Long, Double> e = new Edge<Long, Double>(1L, 2L, 0.5);

// reverse the source and target of this edge
// 反转一条边的两个点
Edge<Long, Double> reversed = e.reverse();

Double weight = e.getValue(); // weight = 0.5
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val e = new Edge(1L, 2L, 0.5)

// reverse the source and target of this edge
// 反转一条边的两个点
val reversed = e.reverse

val weight = e.getValue // weight = 0.5
{% endhighlight %}
</div>
</div>

In Gelly an `Edge` is always directed from the source vertex to the target vertex. A `Graph` may be undirected if for
every `Edge` it contains a matching `Edge` from the target vertex to the source vertex.   
在Gelly中，`Edge`永远从源端点指向目的端点。对一个`Graph`而言，如果每条`Edge` 都对应着另一条从目的端点指向源端点的`Edge`，那么它可能是无向的。  

{% top %}

Graph Creation 创建图
-----------

You can create a `Graph` in the following ways:  
你可以通过如下方法创建一个`Graph`：  

* from a `DataSet` of edges and an optional `DataSet` of vertices:  
* 根据一个由边组成的`DataSet`，可选参数是一个由顶点组成的`DataSet`：  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Vertex<String, Long>> vertices = ...

DataSet<Edge<String, Double>> edges = ...

Graph<String, Long, Double> graph = Graph.fromDataSet(vertices, edges, env);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

val vertices: DataSet[Vertex[String, Long]] = ...

val edges: DataSet[Edge[String, Double]] = ...

val graph = Graph.fromDataSet(vertices, edges, env)
{% endhighlight %}
</div>
</div>

* from a `DataSet` of `Tuple2` representing the edges. Gelly will convert each `Tuple2` to an `Edge`, where the first field will be the source ID and the second field will be the target ID. Both vertex and edge values will be set to `NullValue`.
* 根据一个由表示边的`Tuple2`类组成的`DataSet`。Gelly 将把每个`Tuple2`转换成`Edge`，其中第一个field 将作为源ID，第二个field 将作为目的ID。顶点和边的值都会被置为`NullValue`。  
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Tuple2<String, String>> edges = ...

Graph<String, NullValue, NullValue> graph = Graph.fromTuple2DataSet(edges, env);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

val edges: DataSet[(String, String)] = ...

val graph = Graph.fromTuple2DataSet(edges, env)
{% endhighlight %}
</div>
</div>

* from a `DataSet` of `Tuple3` and an optional `DataSet` of `Tuple2`. In this case, Gelly will convert each `Tuple3` to an `Edge`, where the first field will be the source ID, the second field will be the target ID and the third field will be the edge value. Equivalently, each `Tuple2` will be converted to a `Vertex`, where the first field will be the vertex ID and the second field will be the vertex value:
* 根据一个由`Tuple3`组成的`DataSet`，可选参数是一个由`Tuple2`组成的`DataSet`。这种情况下，Gelly 将把每个`Tuple3`转换成`Edge`，其中第一个field 将成为源ID，第二个field 将成为目的ID，第三个field 将成为边的value。同样地，每个`Tuple2`将被转换为一个`Vertex`，其中第一个field 将成为端点的ID，第二个field 将成为端点的value。  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<Tuple2<String, Long>> vertexTuples = env.readCsvFile("path/to/vertex/input").types(String.class, Long.class);

DataSet<Tuple3<String, String, Double>> edgeTuples = env.readCsvFile("path/to/edge/input").types(String.class, String.class, Double.class);

Graph<String, Long, Double> graph = Graph.fromTupleDataSet(vertexTuples, edgeTuples, env);
{% endhighlight %}

* from a CSV file of Edge data and an optional CSV file of Vertex data. In this case, Gelly will convert each row from the Edge CSV file to an `Edge`, where the first field will be the source ID, the second field will be the target ID and the third field (if present) will be the edge value. Equivalently, each row from the optional Vertex CSV file will be converted to a `Vertex`, where the first field will be the vertex ID and the second field (if present) will be the vertex value. In order to get a `Graph` from a `GraphCsvReader` one has to specify the types, using one of the following methods:
* 根据一个包含边数据的CSV文件，可选参数是一个包含端点数据的CSV文件。这种情况下，Gelly 将把边CSV文件的每一行转换成一个`Edge`，其中第一个field 将成为源ID， 第二个field 将成为目的ID， 第三个field (如果存在的话)将成为边的value。同样地，可选端点CSV文件的每一行将被转换成一个`Vertex`，其中第一个field 将成为端点的ID，第二个field（如果存在的话）将成为端点的value。想从`GraphCsvReader`得到`Graph`，必须用下面的某种方法指定类型：  

- `types(Class<K> vertexKey, Class<VV> vertexValue,Class<EV> edgeValue)`: both vertex and edge values are present.
- `edgeTypes(Class<K> vertexKey, Class<EV> edgeValue)`: the Graph has edge values, but no vertex values.
- `vertexTypes(Class<K> vertexKey, Class<VV> vertexValue)`: the Graph has vertex values, but no edge values.
- `keyType(Class<K> vertexKey)`: the Graph has no vertex values and no edge values.

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// create a Graph with String Vertex IDs, Long Vertex values and Double Edge values
// 生成一个Vertex ID为String 类型、Vertex value为Long 类型，Edge value为Double 类型的图  
Graph<String, Long, Double> graph = Graph.fromCsvReader("path/to/vertex/input", "path/to/edge/input", env)
					.types(String.class, Long.class, Double.class);


// create a Graph with neither Vertex nor Edge values
// 生成一个Vertex 和Edge 都没有value 的图
Graph<Long, NullValue, NullValue> simpleGraph = Graph.fromCsvReader("path/to/edge/input", env).keyType(Long.class);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

val vertexTuples = env.readCsvFile[String, Long]("path/to/vertex/input")

val edgeTuples = env.readCsvFile[String, String, Double]("path/to/edge/input")

val graph = Graph.fromTupleDataSet(vertexTuples, edgeTuples, env)
{% endhighlight %}

* from a CSV file of Edge data and an optional CSV file of Vertex data.
In this case, Gelly will convert each row from the Edge CSV file to an `Edge`.
The first field of the each row will be the source ID, the second field will be the target ID and the third field (if present) will be the edge value.
If the edges have no associated value, set the edge value type parameter (3rd type argument) to `NullValue`.You can also specify that the vertices are initialized with a vertex value.
* 根据一个包含边数据的csv文件，可选参数是一个包含端点数据的csv文件。这种情况下，Gelly 将把边CSV文件的每一行转换成一个`Edge`，其中第一个field 将成为源ID， 第二个field 将成为目的ID， 第三个field (如果存在的话)将成为边的value。如果这条边没有关联的value， 将边的类型参数(第三个类型参数)设为`NullValue`。你也可以指定用某个值初始化端点。  
If you provide a path to a CSV file via `pathVertices`, each row of this file will be converted to a `Vertex`.
The first field of each row will be the vertex ID and the second field will be the vertex value.  
如果通过`pathVertices`提供了CSV 文件的路径，那么文件的每行都会被转换成一个`Vertex`。每行的第一个field 将成为端点的ID， 第二个field 将成为端点的value。  
If you provide a vertex value initializer `MapFunction` via the `vertexValueInitializer` parameter, then this function is used to generate the vertex values.
The set of vertices will be created automatically from the edges input.
If the vertices have no associated value, set the vertex value type parameter (2nd type argument) to `NullValue`.
The vertices will then be automatically created from the edges input with vertex value of type `NullValue`.  
如果通过参数`vertexValueInitializer`提供了端点value的初始化工具`MapFunction` ，那么这个函数可以用来生成端点的值。根据边的输入，可以自动生成端点的集合。如果端点没有关联值，要将端点value的类型参数（第二个类型参数）设为`NullValue`。根据边的输入，会自动生成值类型为`NullValue`的端点集合。  


{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

// create a Graph with String Vertex IDs, Long Vertex values and Double Edge values
val graph = Graph.fromCsvReader[String, Long, Double](
		pathVertices = "path/to/vertex/input",
		pathEdges = "path/to/edge/input",
		env = env)


// create a Graph with neither Vertex nor Edge values
val simpleGraph = Graph.fromCsvReader[Long, NullValue, NullValue](
		pathEdges = "path/to/edge/input",
		env = env)

// create a Graph with Double Vertex values generated by a vertex value initializer and no Edge values
val simpleGraph = Graph.fromCsvReader[Long, Double, NullValue](
        pathEdges = "path/to/edge/input",
        vertexValueInitializer = new MapFunction[Long, Double]() {
            def map(id: Long): Double = {
                id.toDouble
            }
        },
        env = env)
{% endhighlight %}
</div>
</div>


* from a `Collection` of edges and an optional `Collection` of vertices:
* 根据一个由边组成的`Collection`，可选参数是一个由端点组成的`Collection`:  
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

List<Vertex<Long, Long>> vertexList = new ArrayList...

List<Edge<Long, String>> edgeList = new ArrayList...

Graph<Long, Long, String> graph = Graph.fromCollection(vertexList, edgeList, env);
{% endhighlight %}

If no vertex input is provided during Graph creation, Gelly will automatically produce the `Vertex` `DataSet` from the edge input. In this case, the created vertices will have no values. Alternatively, you can provide a `MapFunction` as an argument to the creation method, in order to initialize the `Vertex` values:
如果创建图时没有提供端点数据，Gelly 会根据边的输入自动生成一个`Vertex`的`DataSet`。这种情况下，生成的端点是没有值的。另外，将`MapFunction` 作为构建函数的一个参数传进去，也可以初始化`Vertex`的:  
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// initialize the vertex value to be equal to the vertex ID
// 初始化时，将端点的值设为端点的ID
Graph<Long, Long, String> graph = Graph.fromCollection(edgeList,
				new MapFunction<Long, Long>() {
					public Long map(Long value) {
						return value;
					}
				}, env);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

val vertexList = List(...)

val edgeList = List(...)

val graph = Graph.fromCollection(vertexList, edgeList, env)
{% endhighlight %}

If no vertex input is provided during Graph creation, Gelly will automatically produce the `Vertex` `DataSet` from the edge input. In this case, the created vertices will have no values. Alternatively, you can provide a `MapFunction` as an argument to the creation method, in order to initialize the `Vertex` values:
如果创建图时没有提供端点的数据，Gelly 会根据边的输入自动生成一个`Vertex`的`DataSet`。这种情况下，生成的端点是没有值的。另外，将`MapFunction` 作为构建函数的一个参数传进去，也可初始化`Vertex`的值:  
{% highlight java %}
val env = ExecutionEnvironment.getExecutionEnvironment

// initialize the vertex value to be equal to the vertex ID
// 初始化时，将端点的值设为端点的ID
val graph = Graph.fromCollection(edgeList,
    new MapFunction[Long, Long] {
       def map(id: Long): Long = id
    }, env)
{% endhighlight %}
</div>
</div>

{% top %}

Graph Properties 图的属性
------------

Gelly includes the following methods for retrieving various Graph properties and metrics:
Gelly 提供了一些方法获取图的各种属性:  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// get the Vertex DataSet
// 获取由端点构成的DataSet
DataSet<Vertex<K, VV>> getVertices()

// get the Edge DataSet
// 获取边的DataSet
DataSet<Edge<K, EV>> getEdges()

// get the IDs of the vertices as a DataSet
// 获取由端点的ID构成的DataSet
DataSet<K> getVertexIds()

// get the source-target pairs of the edge IDs as a DataSet
// 获取由边ID构成的source-target pair组成的DataSet
DataSet<Tuple2<K, K>> getEdgeIds()

// get a DataSet of <vertex ID, in-degree> pairs for all vertices
// 获取端点的<端点ID， 入度> pair 组成的DataSet
DataSet<Tuple2<K, LongValue>> inDegrees()

// get a DataSet of <vertex ID, out-degree> pairs for all vertices
// 获取端点的<端点ID， 出度> pair 组成的DataSet
DataSet<Tuple2<K, LongValue>> outDegrees()

// get a DataSet of <vertex ID, degree> pairs for all vertices, where degree is the sum of in- and out- degrees
// 获取端点的<端点ID， 度> pair 组成的DataSet，这里的度 = 入度 + 出度
DataSet<Tuple2<K, LongValue>> getDegrees()

// get the number of vertices
// 获取端点的数量
long numberOfVertices()

// get the number of edges
// 获取边的数量
long numberOfEdges()

// get a DataSet of Triplets<srcVertex, trgVertex, edge>   
// 获取由三元组<srcVertex, trgVertex, edge> 构成的DataSet  
DataSet<Triplet<K, VV, EV>> getTriplets()

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// get the Vertex DataSet
// 获取由端点构成的DataSet
getVertices: DataSet[Vertex[K, VV]]

// get the Edge DataSet
// 获取边的DataSet
getEdges: DataSet[Edge[K, EV]]

// get the IDs of the vertices as a DataSet
// 获取由端点的ID构成的DataSet
getVertexIds: DataSet[K]

// get the source-target pairs of the edge IDs as a DataSet
// 获取由边ID构成的source-target pair组成的DataSet
getEdgeIds: DataSet[(K, K)]

// get a DataSet of <vertex ID, in-degree> pairs for all vertices
// 获取端点的<端点ID， 入度> pair 组成的DataSet  
inDegrees: DataSet[(K, LongValue)]

// get a DataSet of <vertex ID, out-degree> pairs for all vertices
// 获取端点的<端点ID， 出度> pair 组成的DataSet
outDegrees: DataSet[(K, LongValue)]

// get a DataSet of <vertex ID, degree> pairs for all vertices, where degree is the sum of in- and out- degrees
// 获取端点的<端点ID， 度> pair 组成的DataSet，这里的度 = 入度 + 出度
getDegrees: DataSet[(K, LongValue)]

// get the number of vertices
// 获取端点的数量 
numberOfVertices: Long

// get the number of edges
// 获取边的数量 
numberOfEdges: Long

// get a DataSet of Triplets<srcVertex, trgVertex, edge>
// 获取由三元组<srcVertex, trgVertex, edge> 构成的DataSet
getTriplets: DataSet[Triplet[K, VV, EV]]

{% endhighlight %}
</div>
</div>

{% top %}

Graph Transformations 图的变换
-----------------

* <strong>Map</strong>: Gelly provides specialized methods for applying a map transformation on the vertex values or edge values. `mapVertices` and `mapEdges` return a new `Graph`, where the IDs of the vertices (or edges) remain unchanged, while the values are transformed according to the provided user-defined map function. The map functions also allow changing the type of the vertex or edge values.  
* <strong>Map</strong>: Gelly 专门提供了一些方法，用来对端点的值和边的值进行map 变换。`mapVertices`和`mapEdges`返回一个新的`Graph`，它的端点(或者边)的ID保持不变，但是值变成了用户自定义的map 函数所提供的对应值。map 函数也允许改变端点或者边的值的类型。  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
Graph<Long, Long, Long> graph = Graph.fromDataSet(vertices, edges, env);

// increment each vertex value by one
// 把每个端点的值加1
Graph<Long, Long, Long> updatedGraph = graph.mapVertices(
				new MapFunction<Vertex<Long, Long>, Long>() {
					public Long map(Vertex<Long, Long> value) {
						return value.getValue() + 1;
					}
				});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment
val graph = Graph.fromDataSet(vertices, edges, env)

// increment each vertex value by one
val updatedGraph = graph.mapVertices(v => v.getValue + 1)
{% endhighlight %}
</div>
</div>

* <strong>Translate</strong>: Gelly provides specialized methods for translating the value and/or type of vertex and edge IDs (`translateGraphIDs`), vertex values (`translateVertexValues`), or edge values (`translateEdgeValues`). Translation is performed by the user-defined map function, several of which are provided in the `org.apache.flink.graph.asm.translate` package. The same `MapFunction` can be used for all the three translate methods.  
* <strong>Translate</strong>: Gelly 提供专门的方法用来translate 端点和边的ID的类型和值(`translateGraphIDs`)，端点的值(`translateVertexValues`)，或者边的值(`translateEdgeValues`)。Translation 的过程是由用户定义的map 函数完成的，`org.apache.flink.graph.asm.translate` 这个包也提供了一些map 函数。同一个`MapFunction`，在上述三种方法里是通用的。  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
Graph<Long, Long, Long> graph = Graph.fromDataSet(vertices, edges, env);

// translate each vertex and edge ID to a String
// 将每个端点和边的ID translate 成String 类型  
Graph<String, Long, Long> updatedGraph = graph.translateGraphIds(
				new MapFunction<Long, String>() {
					public String map(Long id) {
						return id.toString();
					}
				});

// translate vertex IDs, edge IDs, vertex values, and edge values to LongValue
// 将端点ID，边ID，端点值，边的值 translage 成LongValue 类型  
Graph<LongValue, LongValue, LongValue> updatedGraph = graph
                .translateGraphIds(new LongToLongValue())
                .translateVertexValues(new LongToLongValue())
                .translateEdgeValues(new LongToLongValue())
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment
val graph = Graph.fromDataSet(vertices, edges, env)

// translate each vertex and edge ID to a String
// 将每个端点和边的ID translate 成String 类型  
val updatedGraph = graph.translateGraphIds(id => id.toString)
{% endhighlight %}
</div>
</div>


* <strong>Filter</strong>: A filter transformation applies a user-defined filter function on the vertices or edges of the `Graph`. `filterOnEdges` will create a sub-graph of the original graph, keeping only the edges that satisfy the provided predicate. Note that the vertex dataset will not be modified. Respectively, `filterOnVertices` applies a filter on the vertices of the graph. Edges whose source and/or target do not satisfy the vertex predicate are removed from the resulting edge dataset. The `subgraph` method can be used to apply a filter function to the vertices and the edges at the same time.  
* <strong>Filter</strong>: Filter 变换将用户自定义的filter 函数作用于`Graph`中的顶点/边。`filterOnEdges` 生成原始图的一个sub-graph，只留下那些满足预设条件的边。注意，端点的dataset 将不会变动。对应地，`filterOnVertices` 在图的端点上应用filter。那些源/目的端点不满足vertex条件的边，将从最终的边组成的 dataset中删除。可以使用`subgraph` 方法，同时在端点和边上应用filter 函数。  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Graph<Long, Long, Long> graph = ...

graph.subgraph(
		new FilterFunction<Vertex<Long, Long>>() {
			   	public boolean filter(Vertex<Long, Long> vertex) {
					// keep only vertices with positive values
					return (vertex.getValue() > 0);
			   }
		   },
		new FilterFunction<Edge<Long, Long>>() {
				public boolean filter(Edge<Long, Long> edge) {
					// keep only edges with negative values
					return (edge.getValue() < 0);
				}
		})
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val graph: Graph[Long, Long, Long] = ...

// keep only vertices with positive values
// and only edges with negative values
graph.subgraph((vertex => vertex.getValue > 0), (edge => edge.getValue < 0))
{% endhighlight %}
</div>
</div>

<p class="text-center">
    <img alt="Filter Transformations" width="80%" src="{{ site.baseurl }}/fig/gelly-filter.png"/>
</p>

* <strong>Join</strong>: Gelly provides specialized methods for joining the vertex and edge datasets with other input datasets. `joinWithVertices` joins the vertices with a `Tuple2` input data set. The join is performed using the vertex ID and the first field of the `Tuple2` input as the join keys. The method returns a new `Graph` where the vertex values have been updated according to a provided user-defined transformation function.   
* <strong>Join</strong>: Gelly 提供一些专门的方法，对vertex 和edge 的dataset 与其它输入的dataset 做join 操作。`joinWithVertices` 将端点与输入的一个`Tuple2`组成的dataset 做join。Join 操作使用的key 是端点的ID和`Tuple2` 的第一个field。这个方法返回一个新的`Graph`，其中端点的值已经根据用户定义的转换函数更新过了。  
Similarly, an input dataset can be joined with the edges, using one of three methods. `joinWithEdges` expects an input `DataSet` of `Tuple3` and joins on the composite key of both source and target vertex IDs. `joinWithEdgesOnSource` expects a `DataSet` of `Tuple2` and joins on the source key of the edges and the first attribute of the input dataset and `joinWithEdgesOnTarget` expects a `DataSet` of `Tuple2` and joins on the target key of the edges and the first attribute of the input dataset. All three methods apply a transformation function on the edge and the input data set values.
Note that if the input dataset contains a key multiple times, all Gelly join methods will only consider the first value encountered.  
类似地，使用下面三种方法，输入的dataset 也可以和边做join。`joinWithEdges` 的期望输入是`Tuple3` 组成的 `DataSet`，join 操作发生在源端点和目的端点的ID 形成的组合key 上。`joinWithEdgesOnSource` 的期望输入是`Tuple2` 组成的`DataSet`，join 操作发生在边的源端点和输入的第一个field上。`joinWithEdgesOnTarget` 的期望输入是`Tuple2` 组成的`DataSet`，join 操作发生在边的目的端点和输入的第一个field上。以上的三种方法，都是在边和输入的dataset上应用变换函数。  
注意，输入的dataset 如果包含重复的key，Gelly 中所有的join 方法都只会处理它遇到的第一个 value。  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Graph<Long, Double, Double> network = ...

DataSet<Tuple2<Long, LongValue>> vertexOutDegrees = network.outDegrees();

// assign the transition probabilities as the edge weights
Graph<Long, Double, Double> networkWithWeights = network.joinWithEdgesOnSource(vertexOutDegrees,
				new VertexJoinFunction<Double, LongValue>() {
					public Double vertexJoin(Double vertexValue, LongValue inputValue) {
						return vertexValue / inputValue.getValue();
					}
				});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val network: Graph[Long, Double, Double] = ...

val vertexOutDegrees: DataSet[(Long, LongValue)] = network.outDegrees

// assign the transition probabilities as the edge weights
val networkWithWeights = network.joinWithEdgesOnSource(vertexOutDegrees, (v1: Double, v2: LongValue) => v1 / v2.getValue)
{% endhighlight %}
</div>
</div>

* <strong>Reverse</strong>: the `reverse()` method returns a new `Graph` where the direction of all edges has been reversed.  
* <strong>Reverse</strong>: `reverse()` 反转所有边，然后返回一个新的`Graph`。  

* <strong>Undirected</strong>: In Gelly, a `Graph` is always directed. Undirected graphs can be represented by adding all opposite-direction edges to a graph. For this purpose, Gelly provides the `getUndirected()` method.  
* <strong>Undirected</strong>: Gelly中，所有的`Graph` 永远是有向的。给图中所有边都加上方向相反的边，这样就可以表示无向图。因此，Gelly提供了`getUndirected()`方法。  

* <strong>Union</strong>: Gelly's `union()` method performs a union operation on the vertex and edge sets of the specified graph and the current graph. Duplicate vertices are removed from the resulting `Graph`, while if duplicate edges exist, these will be preserved.
* <strong>Union</strong>: Gelly 的`union()` 方法在指定图和当前图的端点和边的集合上取并集。在得到的`Graph` 中，重复的端点会被删除；如果存在重复边，重复的端点会被保留。  

<p class="text-center">
    <img alt="Union Transformation" width="50%" src="{{ site.baseurl }}/fig/gelly-union.png"/>
</p>

* <strong>Difference</strong>: Gelly's `difference()` method performs a difference on the vertex and edge sets of the current graph and the specified graph.  
* <strong>Difference</strong>：Gelly 的`difference()` 方法在指定图和当前图的端点和边的集合上取差异。  


* <strong>Intersect</strong>: Gelly's `intersect()` method performs an intersect on the edge
 sets of the current graph and the specified graph. The result is a new `Graph` that contains all
 edges that exist in both input graphs. Two edges are considered equal, if they have the same source
 identifier, target identifier and edge value. Vertices in the resulting graph have no
 value. If vertex values are required, one can for example retrieve them from one of the input graphs using
 the `joinWithVertices()` method.  
 * <strong>Intersect</strong>: Gelly 的`intersect()` 方法在指定图和当前图的端点和边的集合上取交集。结果是生成一个新的`Graph`, 包含两个图中都存在的所有边。如果两条边的源 identifier, 目的 identifier，value 都相同，那么就认为它们是相等的。生成的图中，所有的端点都没有value。 如果需要端点的value， 可以通过`joinWithVertices()` 方法从输入图中获取。  
 Depending on the parameter `distinct`, equal edges are either contained once in the resulting
 `Graph` or as often as there are pairs of equal edges in the input graphs.  
 根据`distinct` 参数存在与否，相等边在生成的`Graph` 中出现的次数要么是一次，要么是输入的图中存在的相等边的pair 的数量。  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// create first graph from edges {(1, 3, 12) (1, 3, 13), (1, 3, 13)}
List<Edge<Long, Long>> edges1 = ...
Graph<Long, NullValue, Long> graph1 = Graph.fromCollection(edges1, env);

// create second graph from edges {(1, 3, 13)}
List<Edge<Long, Long>> edges2 = ...
Graph<Long, NullValue, Long> graph2 = Graph.fromCollection(edges2, env);

// Using distinct = true results in {(1,3,13)}
Graph<Long, NullValue, Long> intersect1 = graph1.intersect(graph2, true);

// Using distinct = false results in {(1,3,13),(1,3,13)} as there is one edge pair
Graph<Long, NullValue, Long> intersect2 = graph1.intersect(graph2, false);

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

// create first graph from edges {(1, 3, 12) (1, 3, 13), (1, 3, 13)}
val edges1: List[Edge[Long, Long]] = ...
val graph1 = Graph.fromCollection(edges1, env)

// create second graph from edges {(1, 3, 13)}
val edges2: List[Edge[Long, Long]] = ...
val graph2 = Graph.fromCollection(edges2, env)


// Using distinct = true results in {(1,3,13)}
val intersect1 = graph1.intersect(graph2, true)

// Using distinct = false results in {(1,3,13),(1,3,13)} as there is one edge pair
val intersect2 = graph1.intersect(graph2, false)
{% endhighlight %}
</div>
</div>

-{% top %}

Graph Mutations 图的变化
-----------

Gelly includes the following methods for adding and removing vertices and edges from an input `Graph`:  
Gelly 提供如下方法，增加、删除输入`Graph`的端点或者边:   
<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// adds a Vertex to the Graph. If the Vertex already exists, it will not be added again.
// 添加一个端点。如果端点已经存在，不会重复添加。  
Graph<K, VV, EV> addVertex(final Vertex<K, VV> vertex)

// adds a list of vertices to the Graph. If the vertices already exist in the graph, they will not be added once more.
// 添加一个端点的list。 如果图中已经存在端点，它们最多会被添加一次。  
Graph<K, VV, EV> addVertices(List<Vertex<K, VV>> verticesToAdd)

// adds an Edge to the Graph. If the source and target vertices do not exist in the graph, they will also be added.
// 添加一条边。如果源端点和目的端点在图中不存在，它们也会被添加。  
Graph<K, VV, EV> addEdge(Vertex<K, VV> source, Vertex<K, VV> target, EV edgeValue)

// adds a list of edges to the Graph. When adding an edge for a non-existing set of vertices, the edge is considered invalid and ignored.
// 添加一个边的list。如果在一个不存在的端点集合上添加边，边将被视为不合法，而且会被忽略。  
Graph<K, VV, EV> addEdges(List<Edge<K, EV>> newEdges)

// removes the given Vertex and its edges from the Graph.  
// 从图中移除指定的端点，以及它的边。  
Graph<K, VV, EV> removeVertex(Vertex<K, VV> vertex)

// removes the given list of vertices and their edges from the Graph
// 从图中移除指定的端点的集合，以及它们的边。  
Graph<K, VV, EV> removeVertices(List<Vertex<K, VV>> verticesToBeRemoved)

// removes *all* edges that match the given Edge from the Graph.  
// 移除图中*所有* 与某条给定边match 的边。  
Graph<K, VV, EV> removeEdge(Edge<K, EV> edge)

// removes *all* edges that match the edges in the given list
// 给定一个边的list，移除图中*所有* 与list中的边match 的边。  
Graph<K, VV, EV> removeEdges(List<Edge<K, EV>> edgesToBeRemoved)
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
// adds a Vertex to the Graph. If the Vertex already exists, it will not be added again.  
// 添加一个端点。如果端点已经存在，不会重复添加。  
addVertex(vertex: Vertex[K, VV])

// adds a list of vertices to the Graph. If the vertices already exist in the graph, they will not be added once more.
// 添加一个端点的list。 如果图中已经存在端点，它们最多会被添加一次。  
addVertices(verticesToAdd: List[Vertex[K, VV]])

// adds an Edge to the Graph. If the source and target vertices do not exist in the graph, they will also be added.  
// 添加一条边。如果源端点和目的端点在图中不存在，它们也会被添加。  
addEdge(source: Vertex[K, VV], target: Vertex[K, VV], edgeValue: EV)

// adds a list of edges to the Graph. When adding an edge for a non-existing set of vertices, the edge is considered invalid and ignored.  
// 添加一个边的list。如果在一个不存在的端点集合上添加边，边将被视为不合法，而且会被忽略。  
addEdges(edges: List[Edge[K, EV]])

// removes the given Vertex and its edges from the Graph.  
// 从图中移除指定的端点，以及它的边。  
removeVertex(vertex: Vertex[K, VV])

// removes the given list of vertices and their edges from the Graph
// 从图中移除指定的端点的集合，以及它们的边。  
removeVertices(verticesToBeRemoved: List[Vertex[K, VV]])

// removes *all* edges that match the given Edge from the Graph.  
// 移除图中*所有* 与某条给定边match 的边。  
removeEdge(edge: Edge[K, EV])

// removes *all* edges that match the edges in the given list  
// 给定一个边的list，移除图中*所有* 与list中的边match 的边。  
removeEdges(edgesToBeRemoved: List[Edge[K, EV]])
{% endhighlight %}
</div>
</div>

Neighborhood Methods 邻域方法
-----------

Neighborhood methods allow vertices to perform an aggregation on their first-hop neighborhood.
`reduceOnEdges()` can be used to compute an aggregation on the values of the neighboring edges of a vertex and `reduceOnNeighbors()` can be used to compute an aggregation on the values of the neighboring vertices. These methods assume associative and commutative aggregations and exploit combiners internally, significantly improving performance.  
邻域方法让端点可以在它们first-hop 的邻居上进行聚合。`reduceOnEdges()`方法可以对一个端点的相邻边的值进行聚合，`reduceOnNeighbors()` 方法可以对一个端点的相邻点的值进行聚合。这些方法的聚合具有结合性和交换性，利用了内部的组合，因此极大提升了性能。  
The neighborhood scope is defined by the `EdgeDirection` parameter, which takes the values `IN`, `OUT` or `ALL`. `IN` will gather all in-coming edges (neighbors) of a vertex, `OUT` will gather all out-going edges (neighbors), while `ALL` will gather all edges (neighbors).  
邻域的范围由`EdgeDirection` 这个参数指定，可选值包括`IN`,`OUT`,`ALL`。`IN` 聚合一个端点所有的入边， `OUT` 聚合一个端点所有的出边， `ALL` 聚合一个端点所有的边。  

For example, assume that you want to select the minimum weight of all out-edges for each vertex in the following graph:  
例如，假设你想从图中每个的端点的所有出边中选出最小weight：  

<p class="text-center">
    <img alt="reduceOnEdges Example" width="50%" src="{{ site.baseurl }}/fig/gelly-example-graph.png"/>
</p>

The following code will collect the out-edges for each vertex and apply the `SelectMinWeight()` user-defined function on each of the resulting neighborhoods:  
下面的代码将计算每个端点的出边，并对得到的每个邻域应用自定义的`SelectMinWeight()`函数：  

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Graph<Long, Long, Double> graph = ...

DataSet<Tuple2<Long, Double>> minWeights = graph.reduceOnEdges(new SelectMinWeight(), EdgeDirection.OUT);

// user-defined function to select the minimum weight
// 用户自定义函数，选择最小weight
static final class SelectMinWeight implements ReduceEdgesFunction<Double> {

		@Override
		public Double reduceEdges(Double firstEdgeValue, Double secondEdgeValue) {
			return Math.min(firstEdgeValue, secondEdgeValue);
		}
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val graph: Graph[Long, Long, Double] = ...

val minWeights = graph.reduceOnEdges(new SelectMinWeight, EdgeDirection.OUT)

// user-defined function to select the minimum weight
// 用户自定义函数，选择最小weight
final class SelectMinWeight extends ReduceEdgesFunction[Double] {
	override def reduceEdges(firstEdgeValue: Double, secondEdgeValue: Double): Double = {
		Math.min(firstEdgeValue, secondEdgeValue)
	}
 }
{% endhighlight %}
</div>
</div>

<p class="text-center">
    <img alt="reduceOnEdges Example" width="50%" src="{{ site.baseurl }}/fig/gelly-reduceOnEdges.png"/>
</p>

Similarly, assume that you would like to compute the sum of the values of all in-coming neighbors, for every vertex. The following code will collect the in-coming neighbors for each vertex and apply the `SumValues()` user-defined function on each neighborhood:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Graph<Long, Long, Double> graph = ...

DataSet<Tuple2<Long, Long>> verticesWithSum = graph.reduceOnNeighbors(new SumValues(), EdgeDirection.IN);

// user-defined function to sum the neighbor values
static final class SumValues implements ReduceNeighborsFunction<Long> {

	    	@Override
	    	public Long reduceNeighbors(Long firstNeighbor, Long secondNeighbor) {
		    	return firstNeighbor + secondNeighbor;
	  	}
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val graph: Graph[Long, Long, Double] = ...

val verticesWithSum = graph.reduceOnNeighbors(new SumValues, EdgeDirection.IN)

// user-defined function to sum the neighbor values
final class SumValues extends ReduceNeighborsFunction[Long] {
   	override def reduceNeighbors(firstNeighbor: Long, secondNeighbor: Long): Long = {
    	firstNeighbor + secondNeighbor
    }
}
{% endhighlight %}
</div>
</div>

<p class="text-center">
    <img alt="reduceOnNeighbors Example" width="70%" src="{{ site.baseurl }}/fig/gelly-reduceOnNeighbors.png"/>
</p>

When the aggregation function is not associative and commutative or when it is desirable to return more than one values per vertex, one can use the more general
`groupReduceOnEdges()` and `groupReduceOnNeighbors()` methods.
These methods return zero, one or more values per vertex and provide access to the whole neighborhood.

For example, the following code will output all the vertex pairs which are connected with an edge having a weight of 0.5 or more:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Graph<Long, Long, Double> graph = ...

DataSet<Tuple2<Vertex<Long, Long>, Vertex<Long, Long>>> vertexPairs = graph.groupReduceOnNeighbors(new SelectLargeWeightNeighbors(), EdgeDirection.OUT);

// user-defined function to select the neighbors which have edges with weight > 0.5
static final class SelectLargeWeightNeighbors implements NeighborsFunctionWithVertexValue<Long, Long, Double,
		Tuple2<Vertex<Long, Long>, Vertex<Long, Long>>> {

		@Override
		public void iterateNeighbors(Vertex<Long, Long> vertex,
				Iterable<Tuple2<Edge<Long, Double>, Vertex<Long, Long>>> neighbors,
				Collector<Tuple2<Vertex<Long, Long>, Vertex<Long, Long>>> out) {

			for (Tuple2<Edge<Long, Double>, Vertex<Long, Long>> neighbor : neighbors) {
				if (neighbor.f0.f2 > 0.5) {
					out.collect(new Tuple2<Vertex<Long, Long>, Vertex<Long, Long>>(vertex, neighbor.f1));
				}
			}
		}
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val graph: Graph[Long, Long, Double] = ...

val vertexPairs = graph.groupReduceOnNeighbors(new SelectLargeWeightNeighbors, EdgeDirection.OUT)

// user-defined function to select the neighbors which have edges with weight > 0.5
final class SelectLargeWeightNeighbors extends NeighborsFunctionWithVertexValue[Long, Long, Double,
  (Vertex[Long, Long], Vertex[Long, Long])] {

	override def iterateNeighbors(vertex: Vertex[Long, Long],
		neighbors: Iterable[(Edge[Long, Double], Vertex[Long, Long])],
		out: Collector[(Vertex[Long, Long], Vertex[Long, Long])]) = {

			for (neighbor <- neighbors) {
				if (neighbor._1.getValue() > 0.5) {
					out.collect(vertex, neighbor._2);
				}
			}
		}
   }
{% endhighlight %}
</div>
</div>

When the aggregation computation does not require access to the vertex value (for which the aggregation is performed), it is advised to use the more efficient `EdgesFunction` and `NeighborsFunction` for the user-defined functions. When access to the vertex value is required, one should use `EdgesFunctionWithVertexValue` and `NeighborsFunctionWithVertexValue` instead.

{% top %}

Graph Validation
-----------

Gelly provides a simple utility for performing validation checks on input graphs. Depending on the application context, a graph may or may not be valid according to certain criteria. For example, a user might need to validate whether their graph contains duplicate edges or whether its structure is bipartite. In order to validate a graph, one can define a custom `GraphValidator` and implement its `validate()` method. `InvalidVertexIdsValidator` is Gelly's pre-defined validator. It checks that the edge set contains valid vertex IDs, i.e. that all edge IDs
also exist in the vertex IDs set.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// create a list of vertices with IDs = {1, 2, 3, 4, 5}
List<Vertex<Long, Long>> vertices = ...

// create a list of edges with IDs = {(1, 2) (1, 3), (2, 4), (5, 6)}
List<Edge<Long, Long>> edges = ...

Graph<Long, Long, Long> graph = Graph.fromCollection(vertices, edges, env);

// will return false: 6 is an invalid ID
graph.validate(new InvalidVertexIdsValidator<Long, Long, Long>());

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

// create a list of vertices with IDs = {1, 2, 3, 4, 5}
val vertices: List[Vertex[Long, Long]] = ...

// create a list of edges with IDs = {(1, 2) (1, 3), (2, 4), (5, 6)}
val edges: List[Edge[Long, Long]] = ...

val graph = Graph.fromCollection(vertices, edges, env)

// will return false: 6 is an invalid ID
graph.validate(new InvalidVertexIdsValidator[Long, Long, Long])

{% endhighlight %}
</div>
</div>

{% top %}
