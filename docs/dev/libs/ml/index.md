---
title: "FlinkML - Flink 机器学习库"
nav-id: ml
nav-show_overview: true
nav-title: 机器学习
nav-parent_id: libs
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

FlinkML是Flink内部的机器学习工具库。它是Flink生态圈的新组件，社区成员不断向它贡献新的算法。
FlinkML目标是提供可扩展的机器学习算法，良好的API和工具来使构建端对端的机器学习系统的工作量最小化。
你可以查阅[路线图](https://cwiki.apache.org/confluence/display/FLINK/FlinkML%3A+Vision+and+Roadmap)了解更多关于FlinkML的目标和趋势。

* This will be replaced by the TOC
{:toc}

## 支持的算法

Flink目前支持以下算法：

### 监督学习

* [支持向量机(SVM using CoCoA)](svm.html)
* [多元线性回归](multiple_linear_regression.html)
* [优化框架](optimization.html)

### 非监督学习

* [K最临近算法（KNN）](knn.html)

### 数据处理

* [多项式特征转化（Polynomial Features）](polynomial_features.html)
* [标准化处理（Standard Scaler）](standard_scaler.html)
* [区间缩放处理（Minmax Scaler）](min_max_scaler.html)

### 推荐

* [交替最小二乘法(ALS)](als.html)

### 离群点选择

* [随机离群点选择 (SOS)](sos.html)

### 实用方法

* [距离度量(Distance Metrics)](distance_metrics.html)
* [交叉验证(Cross Validation)](cross_validation.html)

## 开始

你可以通过我们的[快速入门指南](quickstart.html)中的例子了解概况。

如果你想直接实践, 你可需要[创建一个Flink程序]({{ site.baseurl }}/dev/linking_with_flink.html).
然后后，在你项目的`pom.xml`中加入FlinkML的依赖。

{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-ml{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}

需要注意的是FlinkML目前没有编译进二进制文件版。
点击[这里]({{site.baseurl}}/dev/linking.html).了解如何在集群中链接不在二进制文件中的库。

至此，你可以开始你的分析任务了。
下面的代码片段展示了使用FlinkML可以非常简单地训练一个多元线性回归模型。

{% highlight scala %}
// LabeledVector是一个带有标签的特征向量，标签可以使一个类或者是一个实值。
val trainingData: DataSet[LabeledVector] = ...
val testingData: DataSet[Vector] = ...

//可以选用Splitter来将一个数据集分割成训练集和测试集。
val dataSet: DataSet[LabeledVector] = ...
val trainTestData: DataSet[TrainTestDataSet] = Splitter.trainTestSplit(dataSet)
val trainingData: DataSet[LabeledVector] = trainTestData.training
val testingData: DataSet[Vector] = trainTestData.testing.map(lv => lv.vector)

val mlr = MultipleLinearRegression()
  .setStepsize(1.0)
  .setIterations(100)
  .setConvergenceThreshold(0.001)

mlr.fit(trainingData)

// 调试好的模型可以用来做预测。
val predictions: DataSet[LabeledVector] = mlr.predict(testingData)
{% endhighlight %}

## 管道（Pipeline）

FlinkML的一个关键概念是它基于[scikit-learn](http://scikit-learn.org) 的管道机制。
它能帮助你快速建立复杂的数据分析管道，这是每一位数据分析师日常工作中不可或缺的部分。
你可以在[这里](pipelines.html)了解Flink Pipeline的详细情况。

下面的代码片段展示了使用FlinkML可以非常简单地创建数据分析管道。

{% highlight scala %}
val trainingData: DataSet[LabeledVector] = ...
val testingData: DataSet[Vector] = ...

val scaler = StandardScaler()
val polyFeatures = PolynomialFeatures().setDegree(3)
val mlr = MultipleLinearRegression()

// 构建标准化，多项式特征和多元线性回归的管道
val pipeline = scaler.chainTransformer(polyFeatures).chainPredictor(mlr)

// 训练管道
pipeline.fit(trainingData)

// 预测计算
val predictions: DataSet[LabeledVector] = pipeline.predict(testingData)
{% endhighlight %}


通过方法 `chainTransformer`可以将一个`Transformer`和另一个或多个`Transformer`链接在一起。
而通过方法 `chainPredictor`可以将一个 `Predictor` 和一个或多个`Transformer`链接在一起。

## 如何贡献

Flink社区欢迎所有有志提高Flink及其相关库的贡献者。为了方便快速了解贡献的方法，请参看我们的官方[贡献指南]({{site.baseurl}}/dev/libs/ml/contribution_guide.html).
