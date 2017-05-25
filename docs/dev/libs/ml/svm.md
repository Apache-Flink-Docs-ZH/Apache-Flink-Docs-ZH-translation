---
mathjax: include
title: 支持向量机（SVM using CoCoA）
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

* This will be replaced by the TOC
{:toc}

## 描述

使用具有合页损失函数的 CoCoA 算法实现了软间隔 SVM。
此算法解决了如下损失函数最小化问题：

$$\min_{\mathbf{w} \in \mathbb{R}^d} \frac{\lambda}{2} \left\lVert \mathbf{w} \right\rVert^2 + \frac{1}{n} \sum_{i=1}^n l_{i}\left(\mathbf{w}^T\mathbf{x}_i\right)$$

其中 $\mathbf{w}$ 为权值向量，$\lambda$ 为正则化常数，
$$\mathbf{x}_i \in \mathbb{R}^d$$ being the data points and $$l_{i}$$ 为凸损失函数，并依赖于输出分类 $$y_{i} \in \mathbb{R}$$。在当前的实现中正则化项为 $\ell_2$ 范数，损失函数为合页损失函数：

  $$l_{i} = \max\left(0, 1 - y_{i} \mathbf{w}^T\mathbf{x}_i \right)$$

基于前面的选择，问题的定义就等价于软间隔支持向量机(SVM)。最小值通过 SDCA 算法求得，为了让算法在分布式环境下更加高效，COCOA 算法首先在本地的一个数据块上计算若干次 SDCA 迭代，然后再将本地更新合并到有效全局状态中。

全局状态被重新分配到下一轮本地 SDCA 迭代的数据分区，然后执行。 因为只有外层迭代需要网络通信，外层迭代的次数和本地 SDCA 迭代决定了全部的网络消耗。一旦独立的数据分区分布在集群中时，本地 SDCA 是不容易并行的。

算法基于 Jaggi 等人的[相关论文](http://arxiv.org/abs/1409.1458)实现。

## 操作

`SVM` 是一个预测模型（`Predictor`）。
因此，它支持拟合（`fit`）与预测（`predict`）两种操作。

### 拟合

SVM 通过 `LabeledVector` 集合进行训练：

* `fit: DataSet[LabeledVector] => Unit`

### 预测

SVM 会对 FlinkML `Vector` 的所有子类预测其对应的分类标签：

* `predict[T <: Vector]: DataSet[T] => DataSet[(T, Double)]`，其中 `(T, Double)` 对应（原始特征值，预测的分类）

如果想要对模型的预测结果进行评估，可以对已正确分类的样本集做预测。传入 `DataSet[(Vector, Double)]` 返回 `DataSet[(Double, Double)]`。返回结构的首元素为传入参数提供的真值，第二个元素为预测值，可以使用这个`(真值, 预测值)`集合来评估算法的准确率和执行情况：

* `predict: DataSet[(Vector, Double)] => DataSet[(Double, Double)]`

## 参数

SVM 的执行可以通过下面的参数进行控制：

<table class="table table-bordered">
<thead>
  <tr>
    <th class="text-left" style="width: 20%">参数</th>
    <th class="text-center">描述</th>
  </tr>
</thead>

<tbody>
  <tr>
    <td><strong>Blocks</strong></td>
    <td>
      <p>
        设定输入数据被切分后的块数量 每块数据都会执行一个本地SDCA(随机对偶坐标上升)方法 设定值应至少相当于并发总数 若没有指定，那么将使用输入DataSet的并发值作为块的数量 (默认值：<strong>None</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Iterations</strong></td>
    <td>
      <p>
        定义外层方法的最大迭代次数。 也可以认为它定义了SDCA方法应用于块数据的频繁程度。 每次迭代后，本地计算的权值向量更新必须被归纳更新至全局的权值向量 新的权值向量将会在每次迭代开始时广播至所有的SDCA任务 (默认值：<strong>10</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>LocalIterations</strong></td>
    <td>
      <p>
        定义SDCA的最大迭代次数 也可以认为它定义了每次SDCA迭代有多少数据点会从本地数据块中被取出 (默认值：<strong>10</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Regularization</strong></td>
    <td>
      <p>
        定义SVM算法的正则常数 正则常数越大，权值向量的L2范数所起的作用就越小 在使用合页损失函数的情况下，这意味着SVM支持向量的边界间隔会越来越大，哪怕这样的分隔包含了一些错误的分类 (默认值：<strong>1.0</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Stepsize</strong></td>
    <td>
      <p>
      定义更新权值向量的初始步长。 步长越大，每次权重向量值更新的就越多。$\frac{stepsize}{blocks}$ 这个比例实际影响着更新操作。 如果算法变得不稳定，此值需要被调整 (默认值：<strong>1.0</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>ThresholdValue</strong></td>
    <td>
      <p>
       设定一个边界值，若决策函数返回值超过了它，则标记为正类(+1.0)。若决策函数的返回值低于它，标记为负类(-1.0)。 若想要得到原始的决策函数返回值，需要使用 OutputDecisionFunction 参数来表明。(默认值：<strong>0.0</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>OutputDecisionFunction</strong></td>
    <td>
      <p>
        决定 SVM 的预测和评估方法是返回与分离超平面的距离还是二分类的标签值。设定为 true 返回每个输入与超平面的原始距离。 设置为 false 返回二分类标签值（+1.0, -1.0）。(默认值：<strong>false</strong>)
      </p>
    </td>
  </tr>
  <tr>
  <td><strong>Seed</strong></td>
  <td>
    <p>
      定义随机数生成器的种子。 此值决定了 SDCA 方法将会选择哪一个数据点 (默认值：<strong>Random Long Integer</strong>)
    </p>
  </td>
</tr>
</tbody>
</table>

## 例子

{% highlight scala %}
import org.apache.flink.api.scala._
import org.apache.flink.ml.math.Vector
import org.apache.flink.ml.common.LabeledVector
import org.apache.flink.ml.classification.SVM
import org.apache.flink.ml.RichExecutionEnvironment

val pathToTrainingFile: String = ???
val pathToTestingFile: String = ???
val env = ExecutionEnvironment.getExecutionEnvironment

// 从 LibSVM 格式的文件中读取训练数据集
val trainingDS: DataSet[LabeledVector] = env.readLibSVM(pathToTrainingFile)

// 创建 SVM 学习器
val svm = SVM()
  .setBlocks(10)

// 使用 SVM 模型开始训练学习
svm.fit(trainingDS)

// 读取测试数据集
val testingDS: DataSet[Vector] = env.readLibSVM(pathToTestingFile).map(_.vector)

// 对测试数据集进行预测
val predictionDS: DataSet[(Vector, Double)] = svm.predict(testingDS)

{% endhighlight %}
