---
title: "Sample Project using the Java API"
nav-title: Sample Project in Java
nav-parent_id: start
nav-pos: 0
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

通过简单地几步来开始编写你的 Flink Java 程序。


## 要求

唯一的要求是需要安装 Maven 3.0.4 (或者更高)和 Java 7.x (或者更高)


## 创建工程

使用下面其中一个命令来创建Flink Java工程

<ul class="nav nav-tabs" style="border-bottom: none;">
    <li class="active"><a href="#maven-archetype" data-toggle="tab">Use <strong>Maven archetypes</strong></a></li>
    <li><a href="#quickstart-script" data-toggle="tab">Run the <strong>quickstart script</strong></a></li>
</ul>
<div class="tab-content">
    <div class="tab-pane active" id="maven-archetype">
    {% highlight bash %}
    $ mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \{% unless site.is_stable %}
      -DarchetypeCatalog=https://repository.apache.org/content/repositories/snapshots/ \{% endunless %}
      -DarchetypeVersion={{site.version}}
    {% endhighlight %}
        这种方式允许你<strong>为新创建的工程命名</strong>。而且会以交互式地方式询问你为 groupId, artifactId 以及 package 命名。
    </div>
    <div class="tab-pane" id="quickstart-script">
    {% highlight bash %}
{% if site.is_stable %}
    $ curl https://flink.apache.org/q/quickstart.sh | bash
{% else %}
    $ curl https://flink.apache.org/q/quickstart-SNAPSHOT.sh | bash
{% endif %}
    {% endhighlight %}
    </div>
</div>

## 检查工程


运行完上面的命令会在当前工作目录下创建一个新目录。如果你使用了 curl 命令来创建 Flink Java 工程，这个目录的名称是 `quickstart`。否则，就是你输入的 `artifactId` 名字：

{% highlight bash %}
$ tree quickstart/
quickstart/
├── pom.xml
└── src
    └── main
        ├── java
        │   └── org
        │       └── myorg
        │           └── quickstart
        │               ├── BatchJob.java
        │               ├── SocketTextStreamWordCount.java
        │               ├── StreamingJob.java
        │               └── WordCount.java
        └── resources
            └── log4j.properties
{% endhighlight %}

这个工程是一个 Maven 工程, 包含四个类。 StreamingJob 和 BatchJob 是基本的框架程序，SocketTextStreamWordCount 是一个 Streaming 示例；而 WordCount 是一个 Batch 示例。需要注意的是，所有这些类的 main 方法都允许你在开发/测试模式下启动Flink。

我们推荐将这个工程导入到你的IDE中，并进行开发和测试。 如果你用的是 Eclipse，可以使用 [m2e 插件](http://www.eclipse.org/m2e/) 来[导入 Maven 工程](http://books.sonatype.com/m2eclipse-book/reference/creating-sect-importing-projects.html#fig-creating-import)。有些Eclipse发行版 默认嵌入了这个插件，其他的需要你手动去安装。IntelliJ IDE内置就提供了对 Maven 工程的支持。

*给Mac OS X用户的建议*：默认的 JVM 堆内存对 Flink 来说太小了，你必须手动增加内存。这里以 Eclipse 为例，依次选择 `Run Configurations -> Arguments`，然后在 `VM Arguments` 里写入：`-Xmx800m`。

## 编译工程

如果你想要 __编译你的工程__ , 进入到工程所在目录，并输入 `mvn clean install -Pbuild-jar` 命令。 你将会找到  __target/original-your-artifact-id-your-version.jar__ 文件，它可以在任意的 Flink 集群上运行。 还有一个 fat-jar，名为 __target/your-artifact-id-your-version.jar__ ，包含了所有添加到 Maven 工程的依赖。

## 下一步

编写我们自己的程序！

Quickstart 工程包含了一个 WordCount 的实现，也就是大数据处理系统的 Hello World。WordCount 的目标是计算文本中单词出现的频率。比如： 单词 “the” 或者 “house” 在所有的Wikipedia文本中出现了多少次。

__样本输入__:

~~~bash
big data is big
~~~

__样本输出__:

~~~bash
big 2
data 1
is 1
~~~

下面的代码就是 Quickstart 工程的 WordCount 实现，它使用两种操作( FlatMap 和 Reduce )处理了一些文本，并且在标准输出中打印了单词的计数结果。

~~~java
public class WordCount {

  public static void main(String[] args) throws Exception {

    // set up the execution environment
    final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

    // get input data
    DataSet<String> text = env.fromElements(
        "To be, or not to be,--that is the question:--",
        "Whether 'tis nobler in the mind to suffer",
        "The slings and arrows of outrageous fortune",
        "Or to take arms against a sea of troubles,"
        );

    DataSet<Tuple2<String, Integer>> counts =
        // split up the lines in pairs (2-tuples) containing: (word,1)
        text.flatMap(new LineSplitter())
        // group by the tuple field "0" and sum up tuple field "1"
        .groupBy(0)
        .sum(1);

    // execute and print result
    counts.print();
  }
}
~~~

这些操作是在专门的类中定义的，下面是 LineSplitter 类的实现。

~~~java
public static final class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {

  @Override
  public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
    // normalize and split the line
    String[] tokens = value.toLowerCase().split("\\W+");

    // emit the pairs
    for (String token : tokens) {
      if (token.length() > 0) {
        out.collect(new Tuple2<String, Integer>(token, 1));
      }
    }
  }
}
~~~

完整代码参见 {% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/wordcount/WordCount.java "Check GitHub" %}。

有关 API 的完整概述，请参见 [DataStream API]({{ site.baseurl }}/dev/datastream_api.html) 和 [DataSet API]({{ site.baseurl }}/dev/batch/index.html) 章节。 如果你遇到困难，请到 [邮件列表](http://mail-archives.apache.org/mod_mbox/flink-dev/) 里面提问。我们很乐意为你解答。
