---
mathjax: include
title: Cross Validation
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

使用机器学习时，过拟合是一个很常见的问题。你可以理解为算法“记住”了训练集数据，但是不能很好地推断样本数据之外的规律。为了解决过拟合，人们经常会将训练集中的一些数据取出，在训练完成后使用这些取出的数据对算法的性能进行评估。这种方法被称为**交叉验证**。一个好的训练模型应该通过数据的部分子集进行训练，并且对数据的其它部分也同样**有效**。

## 交叉验证策略

对数据进行分割有许多种策略。为了方便使用，FlinkML 提供了以下几种方法：
- 一般分割法
- Holdout 分割法
- K-Fold 分割法
- 多元随机分割法

### 一般分割法

`trainTestSplit` 是最简单的一种分割手段。这个分割器需要输入一个数据集以及一个参数 *fraction*。*feaction* 参数指定了需要分配多少数据集给训练集。它也可以接收 *precise* 与 *seed* 这两个额外的参数。

在默认情况下，此分割器会随机地以 *fraction* 为概率在数据集中抽取子集。如果参数 *precise* 设置为 `true`，分割器还会额外地确认一次训练集与数据集的比值是否与 *fraction* 接近。

此方法会返回一个新的 `TrainTestDataSet` 对象，`.training` 中是训练数据集，`.testing` 中是测试数据集。

### Holdout 分割法

在一些情况下，算法会因为“学习”了测试集的数据造成过拟合。为了避免这种情况，holdout 分割法会创建一个额外的 holdout 数据集（即 *holout* 集）。

算法与平常一样在使用训练集与测试集完成训练之后，会使用 holdout 集做一次额外的最终验证。在理想情况下，holdout 集的预测错误/模型 score 应该和测试集中的结果没有明显的差别。

holdout 分割法为了增加模型没有过拟合的可信度，需要在初始化拟合算法前牺牲一些样本大小。

使用 `trainTestHoldout` 分割器时，原本是 `Double` 类型的 *fraction* 参数会由一个 *farction* 数组代替。数组的第一个元素指定分割多少数据用于训练，第二个元素用于指定测试集，第三个元素指定 holdout 集。数组元素的权值是**相对**来说的，例如数组 `Array(3.0, 2.0, 1.0)` 将会将大约 50% 的数据放入训练集，约 33% 的数据放入测试集，约 17% 的数据放入 holdout 集。

### K-Fold 分割法

在 *k-fold* 策略中，数据集将会被等分成 k 个子集。每个子集都会被设为一次 `.training` (训练)数据集，同时在该次训练中其它的子集会被设为 `.testing` (测试集)。

对于每个训练集来说，算法都会根据它训练同时根据其它的测试集进行测试。当算法对于各个子集训练集都能取得较为连续的评价分数（如预测错误率）时，就可以确定我们使用的方法（包括算法的选择、算法参数、迭代次数等）很难产生过拟合。

Wiki: <a href="https://en.wikipedia.org/wiki/Cross-validation_(statistics)#k-fold_cross-validation">K-Fold 交叉验证</a>

### 多元随机分割法

多元随机分割策略可以看成是 holdout 分割法的一般形式。实际上，`.trainTestHoldoutSplit` 是 `multiRandomSplit` 的一个简单的封装，`multiRandomSplit` 也会将数据包入一个 `trainTestHoldoutDataSet` 对象中。

它们最主要的不同点是 `muiltiRandomSplit` 需要传入任意长度的 *fraction* 数组。例如，它能创建多个 holdout 集。同样，也可以将 `kFoldSplit` 看做 `muiltiRandomSplit` 的封装，它们的不同之处是 `kFoldSplit` 会创建基本等分的子集，而 `muiltiRandomSplit` 会创建任意大小的子集。

`muiltiRandomSplit` 与 holdout 的第二个不同点是 `muiltiRandomSplit` 会返回一个数据集的数组，这个数组的长度与参数传入的 *fraction* 数组长度相同。

## 参数

多种 `Splitter`(分割器) 共有一些参数。

 <table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">参数</th>
      <th class="text-center">类型</th>
      <th class="text-center">描述</th>
      <th class="text-right">可在以下方法中使用</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><code>input</code></td>
      <td><code>DataSet[Any]</code></td>
      <td>需要进行分割的数据集。</td>
      <td>
      <code>randomSplit</code><br>
      <code>multiRandomSplit</code><br>
      <code>kFoldSplit</code><br>
      <code>trainTestSplit</code><br>
      <code>trainTestHoldoutSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>seed</code></td>
      <td><code>Long</code></td>
      <td>
        <p>
          用于根据 seed 生成随机数，将 DataSet 重新排序。
        </p>
      </td>
      <td>
      <code>randomSplit</code><br>
      <code>multiRandomSplit</code><br>
      <code>kFoldSplit</code><br>
      <code>trainTestSplit</code><br>
      <code>trainTestHoldoutSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>precise</code></td>
      <td><code>Boolean</code></td>
      <td>当值为 true 时，会额外进行一步验证，确认数据集的分割结果是否与设定的比率接近。</td>
      <td>
      <code>randomSplit</code><br>
      <code>trainTestSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>fraction</code></td>
      <td><code>Double</code></td>
      <td>`input` 输入值的一部分，用于按照此比例分配第一个数据集或者 <code>.training</code> 训练集。 此参数值必须在 (0,1) 的范围内。</td>
      <td><code>randomSplit</code><br>
        <code>trainTestSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>fracArray</code></td>
      <td><code>Array[Double]</code></td>
      <td>此参数为数组，规定了输出数据集的分配比率（该值不需要和为 1，也无需限定在(0,1)的范围内）。</td>
      <td>
      <code>multiRandomSplit</code><br>
      <code>trainTestHoldoutSplit</code>
      </td>
    </tr>
    <tr>
      <td><code>kFolds</code></td>
      <td><code>Int</code></td>
      <td>此参数规定了将 <code>input</code> 输入数据集分割为多少个子集。</td>
      <td><code>kFoldSplit</code></td>
      </tr>

  </tbody>
</table>

## 例子

{% highlight scala %}
// 输入数据集，类型不一定要是 LabeledVector
val data: DataSet[LabeledVector] = ...

// 进行简单的分割
val dataTrainTest: TrainTestDataSet = Splitter.trainTestSplit(data, 0.6, true)

// 创建 holdout 分割法数据集
val dataTrainTestHO: trainTestHoldoutDataSet = Splitter.trainTestHoldoutSplit(data, Array(6.0, 3.0, 1.0))

// 创建 K-Fold 分割法数据集数组
val dataKFolded: Array[TrainTestDataSet] =  Splitter.kFoldSplit(data, 10)

// 使用多元随机分割创建包含 5 个数据集的数组
val dataMultiRandom: Array[DataSet[T]] = Splitter.multiRandomSplit(data, Array(0.5, 0.1, 0.1, 0.1, 0.1))
{% endhighlight %}
