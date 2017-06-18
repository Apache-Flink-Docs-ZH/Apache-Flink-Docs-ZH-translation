---
title: "Asynchronous I/O for External Data Access"
nav-title: "Async I/O"
nav-parent_id: streaming
nav-pos: 60
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

* ToC
{:toc}

本页介绍了使用Flink的异步I/O API与外部数据存储交互。对于不熟悉异步或事件驱动编程的用户，有关Futures和事件驱动编程的文章可能很有用。

注意：有关异步I/O的设计和实现的详细信息，请参阅提案和设计文件[FLIP-12: Asynchronous I/O Design and Implementation](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=65870673)。


## 异步I/O的需求

当与外部系统交互时（例如使用数据库中存储的数据来丰富流事件），你需要注意与外部系统的通信延迟会影响流应用程序的总体工作。

外部数据库中的数据访问例如在`MapFunction`中，通常意味着**同步**交互：将请求发送到数据库，`MapFunction`等待直到接收到响应。在许多情况下，这种等待占用了function的大部分时间。

与数据库的异步交互意味着单个并行函数实例可以同时处理许多请求并且同时接收响应。这样，可以通过发送其他请求和接收响应来节约等待时间。至少，此时等待时间是通过多个请求进行摊销的。这在大多数情况下可以让流吞吐量更高。

<img src="../../fig/async_io.svg" class="center" width="50%" />

*注意：* 通过提高`MapFunction`的并行度在某些情况下也可以提高吞吐量，但是资源成本很高：具有很多并行的MapFunction实例意味着更多的tasks，threads，Flink内部的网络连接，与数据库的网络连接，buffers以及内部的状态开销。


## 先决条件

如上节所述，为数据库（或key/value存储）实施正确的异步(I/O)需要数据库客户端支持异步请求。许多主流的数据库都提供了这种客户端。

在没有这样的客户端的情况下，你可以通过创建多个客户端并使用线程池处理同步调用来尝试将同步客户端转换为功能有限的并发客户端。不过这种方法通常比合适的异步客户端效率低下。


## 异步I/O API

Flink的异步I/O API允许用户使用数据流的异步请求客户端。API处理与数据流的集成，以及处理顺序，event time，容错等。

假设有一个目标数据库有一个异步客户端，则需要三个部分配合来实现基于数据库异步I/O的stream transformation：

  - `AsyncFunction`的实现以分派请求
  - 一个*callback*，用于取得operation的结果并交给`AsyncCollector`
  - 像应用transformation那样在DataStream上应用异步I/O

以下代码示例说明了基本模式：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// This example implements the asynchronous request and callback with Futures that have the
// interface of Java 8's futures (which is the same one followed by Flink's Future)

/**
 * An implementation of the 'AsyncFunction' that sends requests and sets the callback.
 */
class AsyncDatabaseRequest extends RichAsyncFunction<String, Tuple2<String, String>> {

    /** The database specific client that can issue concurrent requests with callbacks */
    private transient DatabaseClient client;

    @Override
    public void open(Configuration parameters) throws Exception {
        client = new DatabaseClient(host, post, credentials);
    }

    @Override
    public void close() throws Exception {
        client.close();
    }

    @Override
    public void asyncInvoke(final String str, final AsyncCollector<Tuple2<String, String>> asyncCollector) throws Exception {

        // issue the asynchronous request, receive a future for result
        Future<String> resultFuture = client.query(str);

        // set the callback to be executed once the request by the client is complete
        // the callback simply forwards the result to the collector
        resultFuture.thenAccept( (String result) -> {

            asyncCollector.collect(Collections.singleton(new Tuple2<>(str, result)));
         
        });
    }
}

// create the original stream
DataStream<String> stream = ...;

// apply the async I/O transformation
DataStream<Tuple2<String, String>> resultStream =
    AsyncDataStream.unorderedWait(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
/**
 * An implementation of the 'AsyncFunction' that sends requests and sets the callback.
 */
class AsyncDatabaseRequest extends AsyncFunction[String, (String, String)] {

    /** The database specific client that can issue concurrent requests with callbacks */
    lazy val client: DatabaseClient = new DatabaseClient(host, post, credentials)

    /** The context used for the future callbacks */
    implicit lazy val executor: ExecutionContext = ExecutionContext.fromExecutor(Executors.directExecutor())


    override def asyncInvoke(str: String, asyncCollector: AsyncCollector[(String, String)]): Unit = {

        // issue the asynchronous request, receive a future for the result
        val resultFuture: Future[String] = client.query(str)

        // set the callback to be executed once the request by the client is complete
        // the callback simply forwards the result to the collector
        resultFuture.onSuccess {
            case result: String => asyncCollector.collect(Iterable((str, result)));
        }
    }
}

// create the original stream
val stream: DataStream[String] = ...

// apply the async I/O transformation
val resultStream: DataStream[(String, String)] =
    AsyncDataStream.unorderedWait(stream, new AsyncDatabaseRequest(), 1000, TimeUnit.MILLISECONDS, 100)

{% endhighlight %}
</div>
</div>

**重要注意**：`AsyncCollector`在第一次调用`AsyncCollector.collect`时完成。所有后续的`collect`调用会被忽略。

以下两个参数控制异步operations：

  - **Timeout**：定义了异步请求被认为失败时经过的最长时间，该参数可以防止僵死/失败请求。

  - **Capacity**：定义了同时进行的异步请求的数量。尽管异步I/O方式可以显著提高吞吐，但是operator仍有可能成为流应用程序的瓶颈。限制并发请求数量可以确保operator不会累积不断增加的待处理请求，一旦超过容量会触发背压机制。


### 结果的顺序

`AsyncFunction`发出的并发请求的完成顺序未定义，通常是基于哪个请求先完成。为了控制发出结果的顺序，Flink提供了两种模式：

  - **Unordered**：异步请求完成后立即发出结果记录。经过异步I/O operator之后，流中的记录顺序不同于之前。当使用*processing time*作为基本时间特性时，该模式具有最低的延迟和最低的开销。调用`AsyncDataStream.unorderedWait(...)`设置该模式。

  - **Ordered**：在这种情况下会保留流顺序。结果记录发出顺序与触发异步请求的顺序（operators接收记录的顺序）相同。为了实现这一点，operator缓冲结果记录，直到其之前的所有记录都发出（或超时）。这通常在检查点中引入一些额外的延迟和一些开销，因为与无序模式相比，记录或结果会在检查点状态中被维护更长时间。调用`AsyncDataStream.orderedWait(...)`设置该模式。


### Event Time

当流应用程序使用[event time](../event_time.html)时，异步I/O operator会正确处理watermarks。具体如以下两种模式所述：

  - **Unordered**：Watermarks不超过记录，反之亦然，也就是说watermarks建立了一个*有序边界*。记录仅在watermarks之间无序。只有在发出watermark之后，才会发出该watermark之后的记录。反过来说，只有在watermark之前的所有输入结果流过，才会发出该watermark。

    这意味着在存在watermarks的情况下，*unordered*模式引入了一些与*ordered*模式相同的延迟和管理开销。该开销的数量取决于watermark频率。

  - **Ordered**：保留watermarks的顺序，就像保留记录之间的顺序一样。与*processing time*相比，开销没有明显变化。

请注意*Ingestion Time*是*event time*的一种特殊情况，它根据source的processing time自动生成watermarks。


### 容错保证

异步I/O operator提供full exactly-once的容错保证。它将异步请求的记录存储在检查点中，并在故障恢复时恢复/重新触发请求。


### 实现提示

对于具有用于回调的*Executor*（或Scala中的*ExecutionContext*）的*Futures*的实现，我们建议使用`DirectExecutor`，因为回调通常执行最少的工作，并且`DirectExecutor`避免了额外的线程切换开销。回调通常只将结果传递给`AsyncCollector`，并将其添加到输出缓冲区。这样，包括发射记录和与检查点构建这样的重开销逻辑都在专用的线程池中完成。

`DirectExecutor`可以通过`org.apache.flink.runtime.concurrent.Executors.directExecutor()`或者`com.google.common.util.concurrent.MoreExecutors.directExecutor()`得到。


### 警告

**AsyncFunction不能多线程调用**

在这里我们要明确指出的一个常见的误区是，`AsyncFunction`不是以多线程方式调用的。只有一个`AsyncFunction`实例，它被调用来处理流相应分区的每个记录。除非`asyncInvoke(...)`方法快速返回并依赖回调（被客户端依赖），否则结果将是不正确的异步I/O。

例如，如下模式导致阻塞的`asyncInvoke(...)`函数，从而导致异步行为失效：

  - 使用的数据库客户端进行查询时，一直阻塞直到收到返回结果

  - 阻塞/等待异步客户端在`asyncInvoke(...)`方法返回的future-type的对象

