---
mathjax: include
title: 标准化缩放
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

标准化缩放器将会根据用户指定的均值与标准差值对给定数据集进行标准化缩放。
如果用户没有为其指定均值与标准差，标准化缩放器将会根据均值为 0、标准差为 1 对输入数据集进行缩放。
给定输入数据集 $x_1, x_2,... x_n$，它的均值为：

 $$\bar{x} = \frac{1}{n}\sum_{i=1}^{n}x_{i}$$

标准差为：

 $$\sigma_{x}=\sqrt{ \frac{1}{n} \sum_{i=1}^{n}(x_{i}-\bar{x})^{2}}$$

经过缩放后的数据集 $z_1, z_2,...,z_n$ 为：

 $$z_{i}= std \left (\frac{x_{i} - \bar{x}  }{\sigma_{x}}\right ) + mean$$

其中 $\textit{std}$ 与 $\textit{mean}$ 是用户指定的标准差与均值。

## 操作

`StandardScaler` 是一个转换器（`Transformer`），因此它支持拟合（`fit`）与转换（`transform`）两种操作。

### 拟合

StandardScaler 可以在所有`Vector`或`LabeledVector`的子类型上进行训练：

* `fit[T <: Vector]: DataSet[T] => Unit`
* `fit: DataSet[LabeledVector] => Unit`

### 转换

StandardScaler 将 `Vector` 或 `LabeledVector` 的子类型数据集转换到对应的相同类型的数据集：

* `transform[T <: Vector]: DataSet[T] => DataSet[T]`
* `transform: DataSet[LabeledVector] => DataSet[LabeledVector]`

## 参数

标准化缩放器可由下面两个参数进行控制。、

 <table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">参数</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>Mean</strong></td>
      <td>
        <p>
          被缩放数据集的均值。（默认值：<strong>0.0</strong>）
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Std</strong></td>
      <td>
        <p>
          被缩放数据集的标准差。（默认值：<strong>1.0</strong>）
        </p>
      </td>
    </tr>
  </tbody>
</table>

## 例子

{% highlight scala %}
// 创建一个标准化缩放器
val scaler = StandardScaler()
.setMean(10.0)
.setStd(2.0)

// 加载需要进行缩放的数据
val dataSet: DataSet[Vector] = ...

// 计算被缩放数据的均值与标准差
scaler.fit(dataSet)

// 根据前面设定的 均值 =10、标准差 =2.0 对数据集进行缩放
val scaledDS = scaler.transform(dataSet)
{% endhighlight %}
