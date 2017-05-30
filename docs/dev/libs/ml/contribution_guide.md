---
mathjax: include
title: 贡献指南
nav-parent_id: ml
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

Flink 社区非常感谢对 FlinkML 的所有类型的贡献。
FlnikML 为对机器学习有兴趣的人员提供高度且活跃的开源项目来进行工作，这能让可扩展的机器学习得以实现。
以下文档描述了如何对 FlinkML 做贡献。

* This will be replaced by the TOC
{:toc}

## 开始

首先，请阅读 Flink 的[贡献指南](http://flink.apache.org/how-to-contribute.html)。该指南中的所有内容同样能运用于 FlinkML。

## 选择一个主题

如果您在寻找一些新的思路，您可以先查阅我们[路线图](https://cwiki.apache.org/confluence/display/FLINK/FlinkML%3A+Vision+and+Roadmap)，接着你可以查阅[JIRA上未解决问题](https://issues.apache.org/jira/issues/?jql=component%20%3D%20%22Machine%20Learning%20Library%22%20AND%20project%20%3D%20FLINK%20AND%20resolution%20%3D%20Unresolved%20ORDER%20BY%20priority%20DESC)这个列表。
一旦您决定对这些问题的其中一个或一些问题做贡献，您可以将问题归你所有，并追踪您在这个问题上的解决进度。这样的话，其它贡献者会知道不同问题的状态，能够避免重复多余的工作。

如果您已经知道你想要为 FlinkML 贡献什么来让 FlinkML 变得更好。
我们依然建议您为您的想法创建一个 JIRA 问题，在这个问题中告诉 Flink 社区您想要做的事。

## 测试

当提交新的贡献时，还需要提供相关的测试来验证算法的行为。这个测试会在代码发生变化时 (比如重构) 对算法的正确性进行维护。

我们会对单元测试 (unit test) 和集成测试 (integration test) 进行区分，单元测试在 Maven 的测试阶段会被执行，而集成测试会在 Maven 的验证阶段被执行。
Maven 会使用下列命名规则对这二者进行区分：
所有包含以正则表达式 `(IT|Integration)(Test|Suite|Case)` 结尾的类的测试样会被认为是集成测试。
剩余的情况都被认为是单元测试，并只测试对所测试组建所表现的行为。

集成测试是一个需要启动整个 Flink 系统的测试。
为了能合适地进行集成测试，所有的集成测试样例必须是继承 `FlinkTestBase` 特质的类。
该特质会设置正确的 `ExecutionEnvironment` 来让测试能够运行在一个以测试为目的的特殊 `FlinkMiniCluster`上。
因此，一个集成测试应该如下所示：

{% highlight scala %}
class ExampleITSuite extends FlatSpec with FlinkTestBase {
  behavior of "An example algorithm"

  it should "do something" in {
    ...
  }
}
{% endhighlight %}

这个测试风格不一定必须是 `FlatSpec`，它可以是任何其它 Scalatest 的 `Suite` 的子类。
更多详细的信息，请参阅[ScalaTest测试风格](http://scalatest.org/user_guide/selecting_a_style)。

## 文档

当对新的算法有贡献时，添加一些代码注释来描述算法工作的方法和用户可以用来控制算法行为的参数是必要的。
此外，我们鼓励贡献者添加这些信息到线上文档 (online documentation) 上，FlinkML 组件的线上文档可以在 `docs/libs/ml` 找到。

每个新的算法需要用一个 markdown 文件进行描述。
这个文件应该包括至少下列信息点：

1. 这个算法是做什么的？
2. 该算法是如何工作的 (或引用描述)
3. 参数的描述及其默认值
4. 展示如何使用该算法的代码段

为了在 markdown 文件中使用 latex 语法，您需要在 YAML front matter 中引入 `mathjax: include`。

{% highlight java %}
---
mathjax: include
htmlTitle: FlinkML - Example title
title: <a href="../ml">FlinkML</a> - Example title
---
{% endhighlight %}

为了展示数学公式，您需要把您的 latex 代码放在 `$$ ... $$` 内。
对于行内的数学公式，使用 `$ ... $`。
此外，在您的 markdown 文件域内需要引入一些预定义的 latex 命令。
预定义 latex 命令的完整列表，请参阅 `docs/_include/latex_commands.html`。

## 贡献

一旦您实现了算法，并经过充分地测试和注释，您将可以进行 pull request。
关于如何进行 pull request，详情可以参阅[这里](http://flink.apache.org/how-to-contribute.html#contributing-code--documentation)。
