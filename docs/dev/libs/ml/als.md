---
mathjax: include
title: 交替最小二乘法（Alternating Least Squares）
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

交替最小二乘法（ALS）算法将一个给定的R矩阵因式分解为$R$和V两个因子，例如$R \approx U^TV$。未知的行的维度被用作算法的参数，叫做潜在因子。自从矩阵因式分解可以用在推荐系统的场景，$U$和$V$矩阵可以分别称为用户和商品矩阵。用户矩阵的第i列用$u_i$表示，商品矩阵的第i列用$v_i$表示。R矩阵可以用$$(R)_{i,j} = r_{i,j}$$称为评价矩阵。为了找到用户和商品矩阵，如下问题得到了解决：
$$\arg\min_{U,V} \sum_{\{i,j\mid r_{i,j} \not= 0\}} \left(r_{i,j} - u_{i}^Tv_{j}\right)^2 +
\lambda \left(\sum_{i} n_{u_i} \left\lVert u_i \right\rVert^2 + \sum_{j} n_{v_j} \left\lVert v_j \right\rVert^2 \right)$$

$\lambda$作为因式分解的因子，$$n_{u_i}$$作为用户i评过分的商品数量， $$n_{v_j}$$作为商品$j$被评分的次数。这个因式分解方案避免了称作加权$\lambda​$因式分解的过拟合。细节可以在[Zhou et al.](http://dx.doi.org/10.1007/978-3-540-68880-8_32)的论文中找到。
通过修复$U$ 和 $V$矩阵，我们获得可以直接解析的二次形式。问题的解决办法是保证总消耗函数的单调递减。通过对$U$ 或 $V$矩阵的这一步操作，我们逐步的改进了矩阵的因式分解。
R矩阵作为(i,j,r)元组的疏松表示。i为行索引，j为列索引，r为(i,j)位置上的矩阵值。


## 操作

`ALS` 是一个预测模型（`Predictor`）。
因此，它支持拟合（`fit`）与预测（`predict`）两种操作。

### 拟合

ALS用于评价矩阵的疏松表示过程的训练：

* `fit: DataSet[(Int, Int, Double)] => Unit`

### 预测

ALS会对每个元组行列的所有索引进行评分预测：

* `predict: DataSet[(Int, Int)] => DataSet[(Int, Int, Double)]`


## 参数

ALS的实现可以通过下面的参数进行控制：

<table class="table table-bordered">
<thead>
  <tr>
    <th class="text-left" style="width: 20%">参数</th>
    <th class="text-center">描述</th>
  </tr>
</thead>

<tbody>
  <tr>
    <td><strong>NumFactors</strong></td>
    <td>
      <p>
        底层模型中使用的潜在因子数目。等价于计算用户和商品向量的维度。 (默认值：<strong>10</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Lambda</strong></td>
    <td>
      <p>
        因式分解的因子。 该值用于避免过拟合或者由于强生成导致的低性能。 (默认值：<strong>1</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Iterations</strong></td>
    <td>
      <p>
        最大迭代次数。 
        (默认值：<strong>10</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>Blocks</strong></td>
    <td>
      <p>
        设定用户和商品矩阵被分组后的块数量。块越少，发送的冗余数据越少。然而，块越大意味着堆中需要存储的更新消息越大。如果由于OOM导致算法失败，试着降低块的数量。  (默认值：<strong>None</strong>)
      </p>
    </td>
  </tr>  
  <tr>
    <td><strong>Seed</strong></td>
    <td>
      <p>
        用于算法生成初始矩阵的随机种子。
        (默认值：<strong>0</strong>)
      </p>
    </td>
  </tr>
  <tr>
    <td><strong>TemporaryPath</strong></td>
    <td>
      <p>
        导致结果被立即存储到临时目录的路径。如果该值被设定，算法会被分为两个预处理阶段，ALS迭代和计算最后ALS半阶段的处理中阶段。预处理阶段计算给定评分矩阵的OutBlockInformation和InBlockInformation。每步的结果存储在特定的目录。通过将算法分为更多小的步骤，Flink不会在多个算子中分割可用的内存。这让系统可以处理更大的独立消息并提升总性能。  (默认值：<strong>None</strong>)
      </p>
    </td>
  </tr>
</tbody>
</table>

## 例子

// 从CSV文件读取输入数据集
val inputDS: DataSet[(Int, Int, Double)] = env.readCsvFile[(Int, Int, Double)](
  pathToTrainingFile)

// 设定ALS学习器
val als = ALS()
.setIterations(10)
.setNumFactors(10)
.setBlocks(100)
.setTemporaryPath("hdfs://tempPath")

// 通过一个map参数设置其他参数
val parameters = ParameterMap()
.add(ALS.Lambda, 0.9)
.add(ALS.Seed, 42L)

// 计算因式分解
als.fit(inputDS, parameters)

// 从CSV文件读取测试数据
val testingDS: DataSet[(Int, Int)] = env.readCsvFile[(Int, Int)](pathToData)

// 通过矩阵因式分解计算评分
val predictedRatings = als.predict(testingDS)

{% endhighlight %}