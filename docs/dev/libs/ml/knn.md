---
mathjax: include
title: k-Nearest Neighbors Join
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
实现一个精密K最近邻居连接（exact k-nearest neighbors join）算法。假设有一个训练集 $A$ 和一个测试集 $B$，该算法返回

$$
KNNJ(A, B, k) = \{ \left( b, KNN(b, A, k) \right) \text{ where } b \in B \text{ and } KNN(b, A, k) \text{ are the k-nearest points to }b\text{ in }A \}
$$

该暴力方法的目标是计算每个训练点和测试点之间的距离。为了使计算每个训练点之间距离的暴力计算过程更加简化和平滑，本方法使用一个四叉树。该四叉树在训练点的数量上有很好的扩展性，但是在空间维度上的扩展性表现不佳。本算法会自动选择是否采用该四叉树，用户也可以通过设置一个参数来覆盖算法的决定，强制指定是否使用该四叉树。

## 操作

`KNN` 是一个 `Predictor`。
正如所示， 它支持 `fit` 和 `predict` 操作。

### Fit

KNN 通过一个给定的 `Vector` 集来训练:

* `fit[T <: Vector]: DataSet[T] => Unit`

### Predict

KNN 为所有的FlinkML的 `Vector` 的子类预测对应的类别标签：

* `predict[T <: Vector]: DataSet[T] => DataSet[(T, Array[Vector])]`, 这里 `(T, Array[Vector])` 元组对应 (test point, K-nearest training points)

## Parameters

KNN的实现可以由以下参数控制：

   <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">描述</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>K</strong></td>
        <td>
          <p>定义要搜索的最近邻居数量。也就是说，对于每一个测试点，该算法会从训练集中找到K个最近邻居.(默认值: <strong>5</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>DistanceMetric</strong></td>
        <td>
          <p>设置用来计算两点之间距离的度量标准。如果没有指定度量标准，则[[org.apache.flink.ml.metrics.distances.EuclideanDistanceMetric]] 被使用.(默认值: <strong>EuclideanDistanceMetric</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Blocks</strong></td>
        <td>
          <p>设置输入数据将会被切分的块数。该数目至少应该被设置成与并行度相等。如果没有指定块数，则使用作为输入的 [[DataSet]] 的平行度作为块数.(默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>UseQuadTree</strong></td>
        <td>
          <p>一个布尔参数，该参数用来指定是否使用能够对训练集进行分区，并且有可能简化平滑KNN搜索的四叉树。如果该值没有指定，则代码会自动决定是否使用一个四叉树。四叉树的使用在训练点和测试点的数量上有很好的扩展性，但在维度上的扩展性表现不佳.(默认值: <strong>None</strong>)</p>
        </td>
      </tr>
      <tr>
        <td><strong>SizeHint</strong></td>
        <td>
          <p>指定训练集或测试集是否小到能优化KNN搜索所需的向量乘操作。如果训练集小，该值应该是 `CrossHint.FIRST_IS_SMALL`，如果测试集小，则设置成 `CrossHint.SECOND_IS_SMALL`.(默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

## 示例

{% highlight scala %}
import org.apache.flink.api.common.operators.base.CrossOperatorBase.CrossHint
import org.apache.flink.api.scala._
import org.apache.flink.ml.nn.KNN
import org.apache.flink.ml.math.Vector
import org.apache.flink.ml.metrics.distances.SquaredEuclideanDistanceMetric

val env = ExecutionEnvironment.getExecutionEnvironment

// 准备数据
val trainingSet: DataSet[Vector] = ...
val testingSet: DataSet[Vector] = ...

val knn = KNN()
  .setK(3)
  .setBlocks(10)
  .setDistanceMetric(SquaredEuclideanDistanceMetric())
  .setUseQuadTree(false)
  .setSizeHint(CrossHint.SECOND_IS_SMALL)

// 跑 knn join
knn.fit(trainingSet)
val result = knn.predict(testingSet).collect()
{% endhighlight %}

关于使用和不使用四叉树计算KNN的更多细节，参照该介绍: [http://danielblazevski.github.io/](http://danielblazevski.github.io/)
