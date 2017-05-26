---
mathjax: include
title: 优化
# Sub navigation
sub-nav-group: batch
sub-nav-parent: flinkml
sub-nav-title: Optimization
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

* Table of contents
{:toc}

## 数学公式

FlinkML 中的优化框架是一个面向开发人员的包，这个包可以解决机器学习任务中经常遇到的[优化](https://en.wikipedia.org/wiki/Mathematical_optimization)问题。在谈论监督式学习时，这涉及找到一个模型，用一组参数 $w$ 定义，该模型能够在给定一组 $(\x, y)$ 例子的情况下最小化 $f(\wv)$，这里$\x$是一个特征向量，而 $y$ 是一个表征一个回归模型的实数值或分类模型的类别标签的实数。在监督式学习中，需要被最小化的函数通常是以下形式：


\begin{equation} \label{eq:objectiveFunc}
    f(\wv) :=
    \frac1n \sum_{i=1}^n L(\wv;\x_i,y_i) +
    \lambda\, R(\wv)
    \ .
\end{equation}


这里 $L$ 是损失函数，而$R(\wv)$是正则化惩罚 (regularization penalty)。我们使用 $L$ 来衡量一个模型对观察数据的拟合有多好，并且我们使用 $R$ 来影响对一个模型的复杂度损失 (complexity cost)，其中 $\lambda > 0$ 是正则化惩罚。

### 损失函数

在监督式学习中，我们用损失函数来衡量模型的拟合程度，损失函数通过对比模型做出的预测 $p$ 和每个实例的真值 $y$ 来惩罚错误。不同的损失函数可以被回归任务 (比如平方损失) 和分类任务 (比如转折点损失(hinge loss))。

常用的损失函数有：

* 平方损失: $ \frac{1}{2} \left(\wv^T \cdot \x - y\right)^2, \quad y \in \R $
* 转折点损失: $ \max \left(0, 1 - y ~ \wv^T \cdot \x\right), \quad y \in \{-1, +1\} $
* 逻辑损失: $ \log\left(1+\exp\left( -y ~ \wv^T \cdot \x\right)\right), \quad y \in \{-1, +1\}$

### 正则化类型

机器学习中的[正则化] (https://en.wikipedia.org/wiki/Regularization_(mathematics)) 会惩罚需要被评价的模型，为了减少过拟合，最常见的惩罚是 $L_1$ 和 $L_2$ 惩罚，定义如下：

* $L_1$: $R(\wv) = \norm{\wv}_1$
* $L_2$: $R(\wv) = \frac{1}{2}\norm{\wv}_2^2$

$L_2$ 惩罚会惩罚大权重，比起含若干大权重的解决方法，它更倾向含更多小权重的解决方案。
$L_1$ 惩罚能用来归零解决方案中的系数 (solution coefficients)，所以会产生稀疏的解决方案。
$\eqref{eq:objectiveFunc}$ 中的正则化常数 $\lambda$ 决定了用在模型上的正则化数量，该常数经常通过交叉验证模型决定。
关于比较好的正则化类型的对比，可以参阅[这](http://www.robotics.stanford.edu/~ang/papers/icml04-l1l2.pdf)篇由 Andrew Ng 发表的论文
至于什么样的正则化类型会被支持，取决于实际使用的优化算法。

## 随机梯度下降 (Stochastic Gradient Descent)

为了找到一个 (局部) 最小值，梯度下降法会朝着与当前参数 (权重) 相关 $\eqref{eq:objectiveFunc}$ 函数的梯度相反的方向的下降。
为了计算精确的梯度，我们需要对一个数据集里的所有点进行一次传递 (one pass)，让整个过程的计算量增大。
另一个替代的方案是随机梯度下降，在随机梯度下降中我们每一轮迭代都从完整的数据集中采集一个点，并通过在线方式对每个点更新参数。

在小批次的随机梯度下降中，我们则从一个数据集中采样随机数据集，并在每一批次上计算梯度。在算法的每一轮迭代中我们根据从每一个批次中计算得到的梯度，对权重只更新一次。

一个重要的参数是学习率 $\eta$，或者称为步长 (step size)，该参数可由以下列出的五个方法之一决定。
初始步长的设定会对算法的表现有重要的影响。如果想要知道一些实际运用中的指导，清参与 Leon Botou 的 "[Stochastic Gradient Descent Tricks](http://research.microsoft.com/pubs/192769/tricks-2012.pdf)"。

目前随机梯度下降的实现使用所有的分区，这样能让一批次的梯度下降变更加有效。只要一个采集操作被引入到 Flink 中，真实的小批次随机梯度下降算法就会被执行。


### 参数

  随机梯度下降的实现可以由以下参数控制：

   <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">描述</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>正则化惩罚</strong></td>
        <td>
          <p>
            要应用的正则化方程. (默认值: <strong>NoRegularization</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>正则化常数</strong></td>
        <td>
          <p>
            使用的正则化量. (默认值: <strong>0.1</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>损失函数</strong></td>
        <td>
          <p>
            要被优化的损失函数. (默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>迭代数</strong></td>
        <td>
          <p>
            最大迭代次数. (默认值: <strong>10</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>学习率</strong></td>
        <td>
          <p>
            梯度下降法中的初始学习率.
            这个值控制了梯度下降法要沿着梯度的反方向移多远.
            (默认值: <strong>0.1</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>聚合阀值</strong></td>
        <td>
          <p>
            当被设置时，如果目标函数 $\eqref{eq:objectiveFunc}$ 的值的相对变化量小于提供的阀值 $\tau$，则迭代停止
            聚合的标准定义如下：$\left| \frac{f(\wv)_{i-1} - f(\wv)_i}{f(\wv)_{i-1}}\right| < \tau$.
            (默认值: <strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>学习率方法</strong></td>
        <td>
          <p>
            (默认值: <strong>LearningRateMethod.Default</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>衰退值</strong></td>
        <td>
          <p>
            (默认值: <strong>0.0</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

### 正则化

FlinkML 支持含 L1, L2 和无正则化的随机梯度下降。正则化类型必须实现 `RegularizationPenalty` 接口，该接口根据梯度和正则化类型计算新的权重。
下表包含了支持的正则化函数。

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">类别名称</th>
      <th class="text-center">正则化函数 $R(\wv)$</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>无正则化</strong></td>
      <td>$R(\wv) = 0$</td>
    </tr>
    <tr>
      <td><strong>L1正则化</strong></td>
      <td>$R(\wv) = \norm{\wv}_1$</td>
    </tr>
    <tr>
      <td><strong>L2正则化</strong></td>
      <td>$R(\wv) = \frac{1}{2}\norm{\wv}_2^2$</td>
    </tr>
  </tbody>
</table>

### 损失函数

需要被最小化的损失函数需要实现 `LossFunction` 接口，该接口定义了计算损失及其梯度的方法。
任何一个定义了自己 `LossFunction` 或使用  `GenericLossFunction` 类会从一个偏损失函数和一个预测函数构造损失函数。
以下是一个实例：

```Scala
val lossFunction = GenericLossFunction(SquaredLoss, LinearPrediction)
```
支持的偏损失函数请参阅[此处](#partial-loss-function-values).
支持的预测函数请参阅[here](#prediction-function-values).

#### 偏随机函数 (Partial Loss Function) 值 ##

  <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">函数名</th>
        <th class="text-center">描述</th>
        <th class="text-center">损失</th>
        <th class="text-center">损失导数</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>平方损失</strong></td>
        <td>
          <p>
            回归任务中最常用的损失函数.
          </p>
        </td>
        <td class="text-center">$\frac{1}{2} (\wv^T \cdot \x - y)^2$</td>
        <td class="text-center">$\wv^T \cdot \x - y$</td>
      </tr>
      <tr>
        <td><strong>逻辑损失</strong></td>
        <td>
          <p>
            分类任务中最常用的损失函数.
          </p>
        </td>
        <td class="text-center">$\log\left(1+\exp\left( -y ~ \wv^T \cdot \x\right)\right), \quad y \in \{-1, +1\}$</td>
        <td class="text-center">$\frac{-y}{1+\exp\left(y ~ \wv^T \cdot \x\right)}$</td>
      </tr>
      <tr>
        <td><strong>转折点损失</strong></td>
        <td>
          <p>
            可用于分类任务的损失函数.
          </p>
        </td>
        <td class="text-center">$\max \left(0, 1 - y ~ \wv^T \cdot \x\right), \quad y \in \{-1, +1\}$</td>
        <td class="text-center">$\begin{cases}
                                 -y&\text{if } y ~ \wv^T <= 1 \\
                                 0&\text{if } y ~ \wv^T > 1
                                 \end{cases}$</td>
      </tr>
    </tbody>
  </table>

#### 预测函数值 ##

  <table class="table table-bordered">
      <thead>
        <tr>
          <th class="text-left" style="width: 20%">函数名</th>
          <th class="text-center">描述</th>
          <th class="text-center">预测</th>
          <th class="text-center">预测梯度</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><strong>线性预测</strong></td>
          <td>
            <p>
              线性模型比如线性回归和线性分类最常用的函数.
            </p>
          </td>
          <td class="text-center">$\x^T \cdot \wv$</td>
          <td class="text-center">$\x$</td>
        </tr>
      </tbody>
    </table>

#### 有效学习率 ##

这里:

- $j$ 是迭代数

- $\eta_j$ 是每一步 $j$ 的步长

- $\eta_0$ 是初始学习率

- $\lambda$ 是正则化常量

- $\tau$ 是衰退常量, 该常量会使得学习率变成一个递减函数 $j$，也就是说，随着迭代次数的增加，学习率会衰减。衰减的精准率是由函数特定的，请参阅 **反缩放 (Inverse Scaling) ** 和 **Wei Xu 方法 (Wei Xu's Method)** (该方法是 **反缩放** 方法的一个延伸)。

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">函数名</th>
        <th class="text-center">描述</th>
        <th class="text-center">函数</th>
        <th class="text-center">称呼为</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>默认</strong></td>
        <td>
          <p>
            决定步长的默认方法。该方法相当于当 $\tau$ = 0.5 时的 inverse scaling 方法。默认保留该特殊情况来为保持向后兼容性.
          </p>
        </td>
        <td class="text-center">$\eta_j = \eta_0/\sqrt{j}$</td>
        <td class="text-center"><code>LearningRateMethod.Default</code></td>
      </tr>
      <tr>
        <td><strong>常数</strong></td>
        <td>
          <p>
            步长在整个学习任务中保持不变.
          </p>
        </td>
        <td class="text-center">$\eta_j = \eta_0$</td>
        <td class="text-center"><code>LearningRateMethod.Constant</code></td>
      </tr>
      <tr>
        <td><strong>Leon Bottou 方法</strong></td>
        <td>
          <p>
            这是 sklearn 中 <code>"最优的"</code> 方法.
            这个最优初始值必须被提供。
            Sklearn 使用下列启发法: $t_0 = \max(1.0, L^\prime(-\beta, 1.0) / (\alpha \cdot \beta)$.
            其中 $\beta = \sqrt{\frac{1}{\sqrt{\alpha}}}$ 且 $L^\prime(prediction, truth)$ 是损失函数的导数.
          </p>
        </td>
        <td class="text-center">$\eta_j = 1 / (\lambda \cdot (t_0 + j -1)) $</td>
        <td class="text-center"><code>LearningRateMethod.Bottou</code></td>
      </tr>
      <tr>
        <td><strong>反缩放</strong></td>
        <td>
          <p>
            一个决定步长的非常常用的方法.
          </p>
        </td>
        <td class="text-center">$\eta_j = \eta_0 / j^{\tau}$</td>
        <td class="text-center"><code>LearningRateMethod.InvScaling</code></td>
      </tr>
      <tr>
        <td><strong>Wei Xu 方法</strong></td>
        <td>
          <p>
            由 Wei Xu 在 <a href="http://arxiv.org/pdf/1107.2490.pdf">Towards Optimal One Pass Large Scale Learning with Averaged Stochastic Gradient Descent</a> 的方法
          </p>
        </td>
        <td class="text-center">$\eta_j = \eta_0 \cdot (1+ \lambda \cdot \eta_0 \cdot j)^{-\tau} $</td>
        <td class="text-center"><code>LearningRateMethod.Xu</code></td>
      </tr>
    </tbody>
  </table>

### 例子

在随机梯度下降的 Flink 实现中，假如在一个 `DataSet[LabeledVector]` 给定一组实例，并选择性地给定一些初始初始权重，我们可以使用 `GradientDescent.optimize()` 来找到给定数据的最优权重。

用户能提供一个包含 `WeightVector` 元素的初始的 `DataSet[WeightVector]`，或者使用所有集合均为0的默认权重。
一个 `WeightVector` 就是一个含权重的容器类 (container class)，这个类把截距从权重向量中分离出来。这允许我们避免把正则化运用到截距上。



{% highlight scala %}
// 创建一个随机梯度下降实例
val sgd = GradientDescent()
  .setLossFunction(SquaredLoss())
  .setRegularizationPenalty(L1Regularization)
  .setRegularizationConstant(0.2)
  .setIterations(100)
  .setLearningRate(0.01)
  .setLearningRateMethod(LearningRateMethod.Xu(-0.75))


// 获取数据
val trainingDS: DataSet[LabeledVector] = ...

// 根据所提供的数据优化权重
val weightDS = sgd.optimize(trainingDS)
{% endhighlight %}
