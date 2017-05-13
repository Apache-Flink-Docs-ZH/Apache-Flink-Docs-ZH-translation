---
mathjax: include
title: Quickstart Guide
nav-title: Quickstart
nav-parent_id: ml
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

## 介绍

FlinkML 旨在从您的数据中学习一个简单的过程，抽象出来通常带有大数据学习任务的复杂性。 在这个快速入门指南，我们将展示使用 FlinkML 解决简单监督学习问题是多么的简单。 但是首先是一些基础知识，如果你已经熟悉机器学习（ML），请随时跳过接下来的几行。

如 Murphy 所定义的，机器学习（ML）用于检测数据中的模式，并使用这些学习到的模式来预测未来。 我们可以将大多数机器学习（ML）算法分为两大类：监督学习和无监督学习。

* **监督学习** 涉及从一个输入（特征）集合到一个输出集合学习一个功能（映射）。 使用训练集（输入，输出）对来完成学习，我们用来近似映射函数。 监督学习问题进一步分为分类和回归问题。 在分类问题中，我们尝试预测样例属于的类，例如用户是否要点击广告。 另一方面，回归问题是关于预测（实际）数值，通常称为因变量，例如明天的温度是多少。

* **无监督学习** 用来发现数据中的模式和规律。 一个例子是聚类，我们尝试从描述性的特征中发现数据分组。 无监督学习也可用于特征选择，例如通过 [主成分分析（principal components analysis）](https://en.wikipedia.org/wiki/Principal_component_analysis) 进行特征选择。

## 连接 FlinkML

为了在你的项目中使用 FlinkML ，首先你必须建立一个 Flink 程序({{ site.baseurl }}/dev/linking_with_flink.html)。 .
接下来，您必须将 FlinkML 的依赖添加到项目的 `pom.xml` 中：

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-ml{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

## 加载数据

要加载与 FlinkML 一起使用的数据，我们可以使用 Flink 的 ETL 功能，或者使用诸如 LibSVM 格式的格式化数据的专门方法。 对于监督学习问题，通常使用 `LabeledVector` 类来表示 `（标记，特征）` 样例。 `LabeledVector` 对象将具有表示样例特征的 FlinkML `Vector` 成员，以及表示标记的 `Double` 成员，该标记可能是分类问题中的类，也可以是回归问题的因变量。

例如，我们可以使用 Haberman's Survival 数据集，您可以[从 UCI 机器学习数据库下载这个数据集](http://archive.ics.uci.edu/ml/machine-learning-databases/haberman/haberman.data)。 该数据集“包含了对乳腺癌手术患者的存活进行研究的病例”。 数据来自逗号分隔的文件，前3列是特征，最后一列是类，第4列表示患者是否存活5年以上（标记1），或者5年内死亡（标记2）。 您可以查看[ UCI 页面](https://archive.ics.uci.edu/ml/datasets/Haberman%27s+Survival)了解有关数据的更多信息。

我们可以先把数据加载为一个 `DataSet[String]` ：

{% highlight scala %}

import org.apache.flink.api.scala._

val env = ExecutionEnvironment.getExecutionEnvironment

val survival = env.readCsvFile[(String, String, String, String)]("/path/to/haberman.data")

{% endhighlight %}

我们现在可以将数据转换成 `DataSet[LabeledVector]` 。 这将允许我们使用 FlinkML 分类算法的数据集。 我们知道数据集的第四个元素是类标记，其余的是特征，所以我们可以像这样构建 `LabeledVector` 元素：

{% highlight scala %}

import org.apache.flink.ml.common.LabeledVector
import org.apache.flink.ml.math.DenseVector

val survivalLV = survival
  .map{tuple =>
    val list = tuple.productIterator.toList
    val numList = list.map(_.asInstanceOf[String].toDouble)
    LabeledVector(numList(3), DenseVector(numList.take(3).toArray))
  }

{% endhighlight %}

然后，我们可以使用这些数据来训练一个学习器。然而，我们将使用另一个数据集来示例建立学习器；这将让我们展示如何导入其他数据集格式。

**LibSVM 文件**

机器学习数据集的通用格式是 LibSVM 格式，并且可以在 [LibSVM 数据集网站](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/)中找到使用该格式的多个数据集。 FlinkML 提供了通过 `MLUtils` 对象的 `readLibSVM` 函数加载 LibSVM 格式的数据集的实用程序。 您还可以使用 `writeLibSVM` 函数以 LibSVM 格式保存数据集。 我们导入 svmguide1 数据集。 您可以在这里下载[训练集](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/binary/svmguide1)和[测试集](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/binary/svmguide1.t)。 这是一个二进制分类数据集，由 Hsu 等人在他们的实用支持向量机（SVM）指南中使用。 它包含4个数字特征和它的类标记。

我们可以简单地使用下面的代码导入数据集：

{% highlight scala %}

import org.apache.flink.ml.MLUtils

val astroTrain: DataSet[LabeledVector] = MLUtils.readLibSVM(env, "/path/to/svmguide1")
val astroTest: DataSet[(Vector, Double)] = MLUtils.readLibSVM(env, "/path/to/svmguide1.t")
      .map(x => (x.vector, x.label))

{% endhighlight %}

它给了我们两个 `DataSet` 对象，我们会在下面的章节中使用这两个对象来生成一个分类器。

## 分类

一旦我们导入了数据集，我们可以训练一个 `指示器` ，如线性 SVM 分类器。 我们可以为分类器设置多个参数。 这里我们设置 `Blocks` 参数，它用于通过底层CoCoA算法来分割输入。 正则化参数确定应用的 $l_2$ 正则化值，用于避免过拟合。 步长确定权重向量更新到下一个权重向量值的贡献。 此参数设置初始步长。

{% highlight scala %}

import org.apache.flink.ml.classification.SVM

val svm = SVM()
  .setBlocks(env.getParallelism)
  .setIterations(100)
  .setRegularization(0.001)
  .setStepsize(0.1)
  .setSeed(42)

svm.fit(astroTrain)

{% endhighlight %}

我们现在可以对测试集进行预测，并使用 `evaluate` 函数创建（真值，预测）对。

{% highlight scala %}

val evaluationPairs: DataSet[(Double, Double)] = svm.evaluate(astroTest)

{% endhighlight %}

接下来，我们将看到我们如何预处理我们的数据，并使用 FlinkML 的机器学习管道功能。

## 数据预处理和管道

在使用 SVM 分类时，经常鼓励的预处理步骤是将输入特征缩放到 [0,1] 范围，以避免极值特征的影响。 FlinkML 有一些`转换器`，如 `MinMaxScaler` ，用于预处理数据，一个关键特征是将转换器 `转换器` 和`指示器` 链接在一起的能力。 这样我们可以运行相同的转换流程，并且以直接的和类型安全的方式对训练和测试数据进行预测。 您可以在[管道文档](pipelines.html)中阅读更多关于FlinkML管道系统的信息。

我们首先为数据集中的特征创建一个归一化转换，并将其链接到一个新的 SVM 分类器。

{% highlight scala %}

import org.apache.flink.ml.preprocessing.MinMaxScaler

val scaler = MinMaxScaler()

val scaledSVM = scaler.chainPredictor(svm)

{% endhighlight %}

我们现在可以使用我们新创建的管道来对测试集进行预测。
首先，训练缩放器和SVM分类器。
然后测试集的数据将被自动缩放，然后传递给SVM进行预测。

{% highlight scala %}

scaledSVM.fit(astroTrain)

val evaluationPairsScaled: DataSet[(Double, Double)] = scaledSVM.evaluate(astroTest)

{% endhighlight %}

被缩放的输入应该会给我们更好的预测表现。

## 下一步

这个快速入门指南是一个对于 FlinkML 基础概念的介绍，但是你能做更多的事情。我们建议您查看[ FlinkML 文档]({{ site.baseurl }}/dev/libs/ml/index.html)，尝试不同的算法。一个入门的好方法是用自己喜欢的来自于 UCI 机器学习库的数据集和 LibSVM 数据集进行试验。从 [Kaggle](https://www.kaggle.com) 或 [DrivenData](http://www.drivendata.org/) 这样的网站处理一个有趣的问题也是通过与其他数据科学家的竞争来学习的好方法。如果您想提供一些新的算法，请查看我们的[贡献指南](contribution_guide.html)。

**参考文献**

<a name="murphy"></a>[1] Murphy, Kevin P. *Machine learning: a probabilistic perspective.* MIT
press, 2012.

<a name="jaggi"></a>[2] Jaggi, Martin, et al. *Communication-efficient distributed dual
coordinate ascent.* Advances in Neural Information Processing Systems. 2014.

<a name="hsu"></a>[3] Hsu, Chih-Wei, Chih-Chung Chang, and Chih-Jen Lin.
 *A practical guide to support vector classification.* 2003.
