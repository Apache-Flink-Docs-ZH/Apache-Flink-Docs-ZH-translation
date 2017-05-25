---
mathjax: include
title: 多元线性回归
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

 多元线性回归的目标是找到一个最佳拟合输入数据的线性函数。通过输入数据集： $(\mathbf{x}, y)$，多元线性回归将得到一个向量 $\mathbf{w}$，使得残差平方(squared residuals)和最小化：

 $$ S(\mathbf{w}) = \sum_{i=1} \left(y - \mathbf{w}^T\mathbf{x_i} \right)^2$$

将其按照矩阵的方式表示，我们将得到以下公式：

 $$\mathbf{w}^* = \arg \min_{\mathbf{w}} (\mathbf{y} - X\mathbf{w})^2$$

从而得到一个确定解：

  $$\mathbf{w}^* = \left(X^TX\right)^{-1}X^T\mathbf{y}$$

 但是，如果输入数据集过大从而导致无法完全解析所有数据时，可以通过随机梯度下降(SGD)来得到一个近似解。 SGD首先用输入数据集的随机子集求得一个梯度，特定点 $\mathbf{x}_i$ 上的梯度为：

  $$\nabla_{\mathbf{w}} S(\mathbf{w}, \mathbf{x_i}) = 2\left(\mathbf{w}^T\mathbf{x_i} - y\right)\mathbf{x_i}$$

  求解出来的梯度被归一化，缩放公式为： $\gamma = \frac{s}{\sqrt{j}}$，其中 $s$ 为初始步长，$j$ 为当前迭代次数。当前梯度的权重向量减去归一化后的当前梯度值，即得到下一次迭代的权重向量：

  $$\mathbf{w}_{t+1} = \mathbf{w}_t - \gamma \frac{1}{n}\sum_{i=1}^n \nabla_{\mathbf{w}} S(\mathbf{w}, \mathbf{x_i})$$

多元线性回归可以根据输入的SGD迭代次数终止，也可以在达到给定的收敛条件后终止。其中收敛条件为两次迭代之间残差平方和满足：

  $$\frac{S_{k-1} - S_k}{S_{k-1}} < \rho$$

## 操作

`多元线性回归` 是一个预测模型（`Predictor`）。
因此，它支持拟合（`fit`）与预测（`predict`）两种操作。

### 拟合

多元线性回归通过 `LabeledVector` 集合进行训练：

* `fit: DataSet[LabeledVector] => Unit`

### 预测

多元线性回归会对 `Vector` 的所有子类预测其回归值：

* `predict[T <: Vector]: DataSet[T] => DataSet[LabeledVector]`

如果想要对模型的预测结果进行苹果，可以对已包含正确值的样本集做预测。传入 `DataSet[LabeledVector]`，得到每个数据的回归值后返回  `DataSet[(Double, Double)]`。 在每个 `DataSet[(Double, Double)]` 元组中，第一个元素是输入的  `DataSet[LabeledVector]`  中的正确值，第二个元素是对应的预测值。你可以使用 `(正确值, 预测值)` 的对比来评估算法：

* `predict: DataSet[LabeledVector] => DataSet[(Double, Double)]`

## 参数

多元线性回归的实现可以通过下面的参数进行控制：

   <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">描述</th>
      </tr>
    </thead>
    
    <tbody>
      <tr>
        <td><strong>Iterations</strong></td>
        <td>
          <p>
            最大迭代次数。（默认值：<strong>10</strong>）
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Stepsize</strong></td>
        <td>
          <p>
           梯度下降的初始步长。该值确定每次进行梯度下降时，反向移动的距离。调整步长对于算法的稳定收敛及性能非常重要。（默认值：<strong>0.1</strong>）
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>ConvergenceThreshold</strong></td>
        <td>
          <p>
          算法停止迭代的残差平方和变化程度的收敛阈值。（默认值：<strong>None</strong>）
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>LearningRateMethod</strong></td>
        <td>
            <p>
            学习率方法，用于计算每次迭代的有效学习率。Flink ML支持的学习率方法列表见：<a href="optimization.html">learning rate methods</a>。（默认值：<strong>LearningRateMethod.Default</strong>）
            </p>
        </td>
      </tr>
    </tbody>
  </table>

## 例子

{% highlight scala %}
// 创建多元线性回归学习器
val mlr = MultipleLinearRegression()
.setIterations(10)
.setStepsize(0.5)
.setConvergenceThreshold(0.001)

// 载入训练数据及以及测试数据集
val trainingDS: DataSet[LabeledVector] = ...
val testingDS: DataSet[Vector] = ...

// 对提供的数据进行线性拟合
mlr.fit(trainingDS)

// 对测试数据集进行预测
val predictions = mlr.predict(testingDS)
{% endhighlight %}
