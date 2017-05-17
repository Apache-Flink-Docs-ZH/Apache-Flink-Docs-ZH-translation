---
title: "执行计划"
nav-parent_id: execution
nav-pos: 40
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


根据集群中数据大小、机器数目等多种参数，Flink的优化器会自动的为程序选择一种执行策略。在很多场景下，
这将有利于知晓Flink如何执行你的程序。


__计划可视化工具__


Flink自带了执行计划的可视化工具。visualizer工具的HTML文档位于```tools/planVisualizer.html```。
它通过一个JSON文件来表示job的执行计划的，并将执行计划可视化，同时带有执行策略的全部注解。

如下的代码展示了如何打印程序执行计划的JSON：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

...

System.out.println(env.getExecutionPlan());
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment

...

println(env.getExecutionPlan())
{% endhighlight %}
</div>
</div>


为了可视化执行计划，需要做如下步骤：

1. 通过浏览器**打开** ```planVisualizer.html``` ,
2. 将JSON内容**粘贴**到文本，
3. **点击** 绘制按钮.

完成以上步骤后，一个详细的执行计划将会被直观的展示。

<img alt="A flink job execution graph." src="{{ site.baseurl }}/fig/plan_visualizer.png" width="80%">



__Web接口__


Flink 提供了一个web接口用于提交和执行jobs。接口是JobManager监控web接口的一部分，默认运行在8081端口。
通过这个接口提交的job需要在`flink-conf.yaml`文件中设置参数`jobmanager.web.submit.enable: true`。


你可以在job执行前指定程序的参数。可视化的执行计划将使你在job执行前就知晓具体的执行计划。

{% top %}
