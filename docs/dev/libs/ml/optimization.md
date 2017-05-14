---
mathjax: include
title: Optimization
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

在监督式学习中，我们用损失函数来衡量模型的拟合程度，损失函数通过对比模型做出的预测 $p$ 和每个实例的真值 $y$。不同的损失函数可以被回归任务 (比如平方损失) 和分类任务 (比如转折点损失(hinge loss))。

常用的损失函数有：

* 平方损失: $ \frac{1}{2} \left(\wv^T \cdot \x - y\right)^2, \quad y \in \R $
* 转折点损失: $ \max \left(0, 1 - y ~ \wv^T \cdot \x\right), \quad y \in \{-1, +1\} $
* 逻辑损失: $ \log\left(1+\exp\left( -y ~ \wv^T \cdot \x\right)\right), \quad y \in \{-1, +1\}$

### 正则化类型

[Regularization](https://en.wikipedia.org/wiki/Regularization_(mathematics)) in machine learning
imposes penalties to the estimated models, in order to reduce overfitting. The most common penalties
are the $L_1$ and $L_2$ penalties, defined as:

* $L_1$: $R(\wv) = \norm{\wv}_1$
* $L_2$: $R(\wv) = \frac{1}{2}\norm{\wv}_2^2$

The $L_2$ penalty penalizes large weights, favoring solutions with more small weights rather than
few large ones.
The $L_1$ penalty can be used to drive a number of the solution coefficients to 0, thereby
producing sparse solutions.
The regularization constant $\lambda$ in $\eqref{eq:objectiveFunc}$ determines the amount of regularization applied to the model,
and is usually determined through model cross-validation.
A good comparison of regularization types can be found in [this](http://www.robotics.stanford.edu/~ang/papers/icml04-l1l2.pdf) paper by Andrew Ng.
Which regularization type is supported depends on the actually used optimization algorithm.

## 随机梯度下降

In order to find a (local) minimum of a function, Gradient Descent methods take steps in the
direction opposite to the gradient of the function $\eqref{eq:objectiveFunc}$ taken with
respect to the current parameters (weights).
In order to compute the exact gradient we need to perform one pass through all the points in
a dataset, making the process computationally expensive.
An alternative is Stochastic Gradient Descent (SGD) where at each iteration we sample one point
from the complete dataset and update the parameters for each point, in an online manner.

In mini-batch SGD we instead sample random subsets of the dataset, and compute the gradient
over each batch. At each iteration of the algorithm we update the weights once, based on
the average of the gradients computed from each mini-batch.

An important parameter is the learning rate $\eta$, or step size, which can be determined by one of five methods, listed below. The setting of the initial step size can significantly affect the performance of the
algorithm. For some practical tips on tuning SGD see Leon Botou's
"[Stochastic Gradient Descent Tricks](http://research.microsoft.com/pubs/192769/tricks-2012.pdf)".

The current implementation of SGD  uses the whole partition, making it
effectively a batch gradient descent. Once a sampling operator has been introduced in Flink, true
mini-batch SGD will be performed.


### 参数

  The stochastic gradient descent implementation can be controlled by the following parameters:

   <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">Parameter</th>
        <th class="text-center">Description</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>RegularizationPenalty</strong></td>
        <td>
          <p>
            The regularization function to apply. (Default value: <strong>NoRegularization</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>RegularizationConstant</strong></td>
        <td>
          <p>
            The amount of regularization to apply. (Default value: <strong>0.1</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>LossFunction</strong></td>
        <td>
          <p>
            The loss function to be optimized. (Default value: <strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Iterations</strong></td>
        <td>
          <p>
            The maximum number of iterations. (Default value: <strong>10</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>LearningRate</strong></td>
        <td>
          <p>
            Initial learning rate for the gradient descent method.
            This value controls how far the gradient descent method moves in the opposite direction
            of the gradient.
            (Default value: <strong>0.1</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>ConvergenceThreshold</strong></td>
        <td>
          <p>
            When set, iterations stop if the relative change in the value of the objective function $\eqref{eq:objectiveFunc}$ is less than the provided threshold, $\tau$.
            The convergence criterion is defined as follows: $\left| \frac{f(\wv)_{i-1} - f(\wv)_i}{f(\wv)_{i-1}}\right| < \tau$.
            (Default value: <strong>None</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>LearningRateMethod</strong></td>
        <td>
          <p>
            (Default value: <strong>LearningRateMethod.Default</strong>)
          </p>
        </td>
      </tr>
      <tr>
        <td><strong>Decay</strong></td>
        <td>
          <p>
            (Default value: <strong>0.0</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

### 正则化

FlinkML supports Stochastic Gradient Descent with L1, L2 and no regularization. The regularization type has to implement the `RegularizationPenalty` interface,
which calculates the new weights based on the gradient and regularization type.
The following list contains the supported regularization functions.

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Class Name</th>
      <th class="text-center">Regularization function $R(\wv)$</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>NoRegularization</strong></td>
      <td>$R(\wv) = 0$</td>
    </tr>
    <tr>
      <td><strong>L1Regularization</strong></td>
      <td>$R(\wv) = \norm{\wv}_1$</td>
    </tr>
    <tr>
      <td><strong>L2Regularization</strong></td>
      <td>$R(\wv) = \frac{1}{2}\norm{\wv}_2^2$</td>
    </tr>
  </tbody>
</table>

### 损失函数

The loss function which is minimized has to implement the `LossFunction` interface, which defines methods to compute the loss and the gradient of it.
Either one defines ones own `LossFunction` or one uses the `GenericLossFunction` class which constructs the loss function from an outer loss function and a prediction function.
An example can be seen here

```Scala
val lossFunction = GenericLossFunction(SquaredLoss, LinearPrediction)
```

The full list of supported outer loss functions can be found [here](#partial-loss-function-values).
The full list of supported prediction functions can be found [here](#prediction-function-values).

#### 偏随机函数 (Partial Loss Function) 值 ##

  <table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">Function Name</th>
        <th class="text-center">Description</th>
        <th class="text-center">Loss</th>
        <th class="text-center">Loss Derivative</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>SquaredLoss</strong></td>
        <td>
          <p>
            Loss function most commonly used for regression tasks.
          </p>
        </td>
        <td class="text-center">$\frac{1}{2} (\wv^T \cdot \x - y)^2$</td>
        <td class="text-center">$\wv^T \cdot \x - y$</td>
      </tr>
      <tr>
        <td><strong>LogisticLoss</strong></td>
        <td>
          <p>
            Loss function used for classification tasks.
          </p>
        </td>
        <td class="text-center">$\log\left(1+\exp\left( -y ~ \wv^T \cdot \x\right)\right), \quad y \in \{-1, +1\}$</td>
        <td class="text-center">$\frac{-y}{1+\exp\left(y ~ \wv^T \cdot \x\right)}$</td>
      </tr>
      <tr>
        <td><strong>HingeLoss</strong></td>
        <td>
          <p>
            Loss function used for classification tasks.
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
          <th class="text-left" style="width: 20%">Function Name</th>
          <th class="text-center">Description</th>
          <th class="text-center">Prediction</th>
          <th class="text-center">Prediction Gradient</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td><strong>LinearPrediction</strong></td>
          <td>
            <p>
              The function most commonly used for linear models, such as linear regression and
              linear classifiers.
            </p>
          </td>
          <td class="text-center">$\x^T \cdot \wv$</td>
          <td class="text-center">$\x$</td>
        </tr>
      </tbody>
    </table>

#### 有效学习率 ##

Where:

- $j$ is the iteration number

- $\eta_j$ is the step size on step $j$

- $\eta_0$ is the initial step size

- $\lambda$ is the regularization constant

- $\tau$ is the decay constant, which causes the learning rate to be a decreasing function of $j$, that is to say as iterations increase, learning rate decreases. The exact rate of decay is function specific, see **Inverse Scaling** and **Wei Xu's Method** (which is an extension of the **Inverse Scaling** method).

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">Function Name</th>
        <th class="text-center">Description</th>
        <th class="text-center">Function</th>
        <th class="text-center">Called As</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td><strong>Default</strong></td>
        <td>
          <p>
            The function default method used for determining the step size. This is equivalent to the inverse scaling method for $\tau$ = 0.5.  This special case is kept as the default to maintain backwards compatibility.
          </p>
        </td>
        <td class="text-center">$\eta_j = \eta_0/\sqrt{j}$</td>
        <td class="text-center"><code>LearningRateMethod.Default</code></td>
      </tr>
      <tr>
        <td><strong>Constant</strong></td>
        <td>
          <p>
            The step size is constant throughout the learning task.
          </p>
        </td>
        <td class="text-center">$\eta_j = \eta_0$</td>
        <td class="text-center"><code>LearningRateMethod.Constant</code></td>
      </tr>
      <tr>
        <td><strong>Leon Bottou's Method</strong></td>
        <td>
          <p>
            This is the <code>'optimal'</code> method of sklearn.
            The optimal initial value $t_0$ has to be provided.
            Sklearn uses the following heuristic: $t_0 = \max(1.0, L^\prime(-\beta, 1.0) / (\alpha \cdot \beta)$
            with $\beta = \sqrt{\frac{1}{\sqrt{\alpha}}}$ and $L^\prime(prediction, truth)$ being the derivative of the loss function.
          </p>
        </td>
        <td class="text-center">$\eta_j = 1 / (\lambda \cdot (t_0 + j -1)) $</td>
        <td class="text-center"><code>LearningRateMethod.Bottou</code></td>
      </tr>
      <tr>
        <td><strong>Inverse Scaling</strong></td>
        <td>
          <p>
            A very common method for determining the step size.
          </p>
        </td>
        <td class="text-center">$\eta_j = \eta_0 / j^{\tau}$</td>
        <td class="text-center"><code>LearningRateMethod.InvScaling</code></td>
      </tr>
      <tr>
        <td><strong>Wei Xu's Method</strong></td>
        <td>
          <p>
            Method proposed by Wei Xu in <a href="http://arxiv.org/pdf/1107.2490.pdf">Towards Optimal One Pass Large Scale Learning with
            Averaged Stochastic Gradient Descent</a>
          </p>
        </td>
        <td class="text-center">$\eta_j = \eta_0 \cdot (1+ \lambda \cdot \eta_0 \cdot j)^{-\tau} $</td>
        <td class="text-center"><code>LearningRateMethod.Xu</code></td>
      </tr>
    </tbody>
  </table>

### 例子

In the Flink implementation of SGD, given a set of examples in a `DataSet[LabeledVector]` and
optionally some initial weights, we can use `GradientDescent.optimize()` in order to optimize
the weights for the given data.

The user can provide an initial `DataSet[WeightVector]`,
which contains one `WeightVector` element, or use the default weights which are all set to 0.
A `WeightVector` is a container class for the weights, which separates the intercept from the
weight vector. This allows us to avoid applying regularization to the intercept.



{% highlight scala %}
// Create stochastic gradient descent solver
val sgd = GradientDescent()
  .setLossFunction(SquaredLoss())
  .setRegularizationPenalty(L1Regularization)
  .setRegularizationConstant(0.2)
  .setIterations(100)
  .setLearningRate(0.01)
  .setLearningRateMethod(LearningRateMethod.Xu(-0.75))


// Obtain data
val trainingDS: DataSet[LabeledVector] = ...

// Optimize the weights, according to the provided data
val weightDS = sgd.optimize(trainingDS)
{% endhighlight %}
