---
mathjax: include
title: Polynomial Features
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

多项式特征转换器 (polynomial features transformer)  能把一个向量映射到一个自由度为 $d$ 的多项式特征空间。
输入特征的维度决定了多项式因子的个数，多项式因子的值就是每个向量的项。
给定一个向量 $(x, y, z, \ldots)^T$，那么被映射的特征向量为：

$$\left(x, y, z, x^2, xy, y^2, yz, z^2, x^3, x^2y, x^2z, xy^2, xyz, xz^2, y^3, \ldots\right)^T$$

Flink 的实现以自由度降序的方式给多项式排序。

如果一个向量为 $\left(3,2\right)^T$，自由度为 3 的多项式特征向量为：

 $$\left(3^3, 3^2\cdot2, 3\cdot2^2, 2^3, 3^2, 3\cdot2, 2^2, 3, 2\right)^T$$

该转换器能够前置于所有 `Transformer` 和 `Predictor` 的实现，这些实现的输入需要是 `LabeledVector` 类或者 `Vector` 的子类。

## 操作

`PolynomialFeatures` 是一个 `Transformer` (转换器)。
因此，它支持 `fit` (拟合) 和 `transform` (转换) 操作。

### 拟合

PolynomialFeatures 并不对数据进行训练，因此它支持所有类型的输入数据。

### 转换

PolynomialFeatures 把所有 `Vector` 和 `LabeledVector` 的子类转换成各自的类型，如下所示：

* `transform[T <: Vector]: DataSet[T] => DataSet[T]`
* `transform: DataSet[LabeledVector] => DataSet[LabeledVector]`

## 参数

多项式特征转换器可以由以下参数控制：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">描述</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>自由度</strong></td>
        <td>
          <p>
            最大多项式自由度.
            (默认值: <strong>10</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

## 例子

{% highlight scala %}
// 获取训练数据集
val trainingDS: DataSet[LabeledVector] = ...

// 设置一个自由度为 3 的多项式特征转换器
val polyFeatures = PolynomialFeatures()
.setDegree(3)

// 设置一个复线性回归学习器
val mlr = MultipleLinearRegression()

// 通过参数映射 (ParameterMap) 控制学习器
val parameters = ParameterMap()
.add(MultipleLinearRegression.Iterations, 20)
.add(MultipleLinearRegression.Stepsize, 0.5)

// 创建管道 (pipeline) PolynomialFeatures -> MultipleLinearRegression
val pipeline = polyFeatures.chainPredictor(mlr)

// 训练模型
pipeline.fit(trainingDS)
{% endhighlight %}
