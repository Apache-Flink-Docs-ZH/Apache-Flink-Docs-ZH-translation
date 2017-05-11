---
mathjax: include
title: Distance Metrics
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

对于不同类型的分析，采用不同类型的距离度量标准是很方便的。Flink ML 为许多标准的距离度量标准提供了内置的实现。
你能通过实现 `DistanceMetric` 特质来创造自定义的距离度量标准。

## 内置实现

目前, FlinkML 支持以下度量标准：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">Metric</th>
        <th class="text-center">Description</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>欧式距离</strong></td>
        <td>
          $$d(\x, \y) = \sqrt{\sum_{i=1}^n \left(x_i - y_i \right)^2}$$
        </td>
      </tr>
      <tr>
        <td><strong>平方欧式距离</strong></td>
        <td>
          $$d(\x, \y) = \sum_{i=1}^n \left(x_i - y_i \right)^2$$
        </td>
      </tr>
      <tr>
        <td><strong>余弦相似度</strong></td>
        <td>
          $$d(\x, \y) = 1 - \frac{\x^T \y}{\Vert \x \Vert \Vert \y \Vert}$$
        </td>
      </tr>
      <tr>
        <td><strong>切比雪夫距离</strong></td>
        <td>
          $$d(\x, \y) = \max_{i}\left(\left \vert x_i - y_i \right\vert \right)$$
        </td>
      </tr>
      <tr>
        <td><strong>曼哈顿距离</strong></td>
        <td>
          $$d(\x, \y) = \sum_{i=1}^n \left\vert x_i - y_i \right\vert$$
        </td>
      </tr>
      <tr>
        <td><strong>闵式距离</strong></td>
        <td>
          $$d(\x, \y) = \left( \sum_{i=1}^{n} \left( x_i - y_i \right)^p \right)^{\rfrac{1}{p}}$$
        </td>
      </tr>
      <tr>
        <td><strong>谷本距离</strong></td>
        <td>
          $$d(\x, \y) = 1 - \frac{\x^T\y}{\Vert \x \Vert^2 + \Vert \y \Vert^2 - \x^T\y}$$
          with $\x$ and $\y$ being bit-vectors
        </td>
      </tr>
    </tbody>
  </table>

## 自定义实现

你能通过实现 `DistanceMetric` 特质来创造你自己的距离度量标准。

{% highlight scala %}
class MyDistance extends DistanceMetric {
  override def distance(a: Vector, b: Vector) = ... // your implementation for distance metric
}

object MyDistance {
  def apply() = new MyDistance()
}

val myMetric = MyDistance()
{% endhighlight %}
