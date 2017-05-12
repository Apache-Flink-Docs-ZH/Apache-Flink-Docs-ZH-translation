---
mathjax: include
title: Looking under the hood of pipelines
nav-title: Pipelines
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

## 简介

把 transformer 和 predictor 链接起来的能力对任何机器学习库都是一个非常重要的特性。在 FlinkML 中，我们希望在提供一个直观的API的同时，能够充分利用 Scala 语言的能力来为我们的 pipelines 提供类型安全的实现。我们希望能够实现的是让API的使用变得简单轻松，让使用者准备 (pre-flight) 时 (工作开始运行之前) 避免类型错误，并且消除在需要长期运行的工作提交后由于数据转换操作错误引起的失败情形，而这类错误在机器学习 pipeline 中是经常发生的。

在本指南中，我们将会描述在 FlinkML 中实现可链接的 transformers 和 predictors 时所采用的选择，并且为开发人员提供关于如何充分使用 pipeline 特性来创建自己的算法的指导。

## "是什么" 和 "为什么"

机器学习中的 pipeline 是什么？在讨论机器学习时，pipeline 可以认为是一系列操作的链接，这些操作以某些数据作为输入，再对这些输入数据进行转换操作，最后输出转换之后的数据，这些转换后的数据既可以被用作 predictor 函数，比如一个学习模型，的输入（特征），也可以仅仅作为被某些其它任务所使用的输出。终端 learner 当然也可以作为 pipeline 的一部分。机器学习中的 pipeline 通常是复杂的操作集合([深入解释](http://research.google.com/pubs/pub43146.html))，并且可以作为终端对终端学习系统的错误源。 

机器学习中 pipeline 的目的是创建一个可以管理由操作链引起的复杂问题的框架。Pipeline 应让开发人员能简易地定义使用在训练数据上的转换操作链，这样才能创建训练学习模型时所需的终端特性 (end features)，并且对没有标签的（测试）数据简易地执行相同的转换操作集。Pipelines 也应简化在这些操作链上进行的交叉验证的模型选择。

最后，pipeline 中的链是连贯相扣的，我们通过确保这些连贯的链能"前后匹配"来避免高代价的类型错误。因为 pipline 中的每一步都可能是计算量繁重的操作，所以我们只有在确定 pipeline 中所有的输入/输出对都能"匹配"时，才会运行一个 pipeline 工作。

## FlinkML 中的 Pipelines

FlinkML 中的 pipeline 构建模块请参阅 `ml.pipeline` 包。FlinkML 的 API 受 [sklearn](http://scikit-learn.org) 启发，这意味着我们有 `Estimator`, `Transformer` 和 `Predictor` 三个接口。想要更加深入地了解 sklearn API 是如何设计的读者，请参阅 [此](http://arxiv.org/abs/1309.0238) 论文。简单地说，`Estimator` 作为基类被`Transformer` 和 `Predictor`继承。`Estimator` 中定义了一个 `fit` 方法，`Transformer` 中定了一个 `transform` 方法，而 `Predictor` 中定义了一个`predict` 方法。

`Estimator` 中的 `fit` 方法对模型进行实质上的训练，比如在一个线性回归任务中找到合适的权重，或者在特征缩放中找到正确的平均值和标准差。对于 `Transformer`, 正如其名所示，任何实现了 `Transformer` 的类都可以实行转换操作，比如 [复线性回归]({{site.baseurl}}/dev/libs/ml/multiple_linear_regression.html)。`Predictor` 的实现类可以学习算法，比如 [复线性回归]({{site.baseurl}}/dev/libs/ml/multiple_linear_regression.html)。Pipeline 可以通过链接若干个 Transformers 创建，pipeline 中的最后一环可以是一个 Predictor 或者是 Transformer，如果一个 pipeline 以 Predictor 结束, 则无法进行下一步链接。下面的例子展示如何构建一个 pipeline:

{% highlight scala %}
// 训练数据
val input: DataSet[LabeledVector] = ...
// 测试数据
val unlabeled: DataSet[Vector] = ...

val scaler = StandardScaler()
val polyFeatures = PolynomialFeatures()
val mlr = MultipleLinearRegression()

// 构建 pipeline
val pipeline = scaler
  .chainTransformer(polyFeatures)
  .chainPredictor(mlr)

// 训练 pipeline (StandardScaler 和 MultipleLinearRegression)
pipeline.fit(input)

// 对测试集进行预测
val predictions: DataSet[LabeledVector] = pipeline.predict(unlabeled)

{% endhighlight %}

正如我们提到的，FlinkML 的 pipeline 是类型安全的。
如果我们尝试把输出类型为 `A` 的 transformer 链接到另一个输入类型为 `B` 的 transformer，若 `A` != `B` 我们会在编译时得到一个错误。FlinkML 中这种类型安全是通过 Scala 语言的隐式特性实现的。

### Scala 的隐式

如果你对 Scala 语言的隐式特性不熟悉，我们推荐 [这篇摘录](https://www.artima.com/pins1ed/implicit-conversions-and-parameters.html)，此摘录取自 Martin Odersky 的 "Programming in Scala"。 简单地说，Scala 中的隐式转换通过一个类型到另一个类型的转换来允许特定的多态，而隐式值则通过隐式参数为编译器提供默认值，这些值可在函数调用时被使用。隐式转换和隐式参数的组合让 transform 和 predict 操作能用类型安全的方式得以实现。

### 操作

正如我们提到的，`Estimator` 特质（抽象类）定义了一个 `fit` 方法。该方法有两个参数列表 (i.e. 是一个[柯里化函数](http://docs.scala-lang.org/tutorials/tour/currying.html))。第一个参数列表接收输入（训练）集 `DataSet` 和提供给 estimator 的参数。第二个参数列表接收一个 `implicit`（隐式）参数，该参数的类型是 `FitOperation`。`FitOperation` 也是一个定义了 `fit` 方法的类，该方法应实现具体的 estimator 训练的实际逻辑。`Estimator` 中的 `fit` 方法本质上是一个 `FitOperation` 中的 `fit` 方法的包装。同理，`Predictor` 的 `predict` 方法和 `transform` 的 `Transform` 方法也通过各自的操作类(operation class)，用相似的方法设计。

这些方法中的操作对象(operation object)均由隐式参数提供。Scala 会在一个类型的伴生对象中[寻找该隐式](http://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html)，因此所有实现了这些接口的类应在其伴生对象中，以隐式对象的形式提供这些对象。

举个例子，我们可以查看 `StandardScaler` 类。`StandardScaler` 继承了 `Transformer`，所以它能调用 `fit` 和 `transform` 方法，这两个方法需要 `FitOperation` 和 `TransformOperation` 作为隐式参数，分别给 `fit` 和 `transform` 方法。 这两个隐式参数在 `StandardScaler` 的伴生对象中通过 `transformVectors` and `fitVectorStandardScaler` 提供：

{% highlight scala %}
class StandardScaler extends Transformer[StandardScaler] {
  ...
}

object StandardScaler {

  ...

  implicit def fitVectorStandardScaler[T <: Vector] = new FitOperation[StandardScaler, T] {
    override def fit(instance: StandardScaler, fitParameters: ParameterMap, input: DataSet[T])
      : Unit = {
        ...
      }

  implicit def transformVectors[T <: Vector: VectorConverter: TypeInformation: ClassTag] = {
      new TransformOperation[StandardScaler, T, T] {
        override def transform(
          instance: StandardScaler,
          transformParameters: ParameterMap,
          input: DataSet[T])
        : DataSet[T] = {
          ...
        }

}

{% endhighlight %}

注意到 `StandardScaler` 并**不会**覆写 `Estimator` 中的 `fit` 方法或 `transform` 中的 `Transformer` 方法。 而它对 `FitOperation` 和 `TransformOperation` 的实现复写他们各自的 `fit` 和 `transform` 方法, 这两个方法分别被 `Estimator` 的 `fit` and `Transformer` 的 `transform` 方法调用。 相似地, 一个实现了 `Predictor` 的类应在它的伴生对象内定义一个隐式 `PredictOperation` 对象。

#### 类型和类型安全

除了我们上面所列出的 `fit` 和 `transform` 操作，`StandardScaler` 还为 `LabeledVector` 类的输入提供了 `fit` 和 `transform` 操作。这允许我们在输入是有标签或没标签时都能使用该算法，并且会根据我们所给的输入的类型，无论要进行拟合操作还是转换操作，自动使用。编译器会根据输入的类型选择正确的隐式操作。

如果我们尝试在调用 `fit` 或 `transform` 方法时使用不支持的类型，我们会在工作开始前获得一个运行时错误。尽管这些错误也有可能在编译时就被捕获，但是我们能够提供给使用者的错误信息所含的信息量就少得多，因此我们选择在运行时抛出异常。

### 链接

链接是通过调用实现了 `Transformer` 的类的对象中的 `chainTransformer` 或 `chainPredictor`来实现。这些方法分别返回一个 `ChainedTransformer` 或 `ChainedPredictor` 对象。正如我们提到的，`ChainedTransformer` 对象能够进行进一步的链接，而 `ChainedPredictor` 对象则不可以。这些类对由一连窜或一个 transformer 和 一个 predictor 组成的搭配进行拟合 (fit)，转换 (transform) 和预测 (predict) 操作。如果连接的长度大于二，他们会递归地运行，因为每个 `ChainedTransformer` 都定义了一个可以与更多 transformer  或一个 predictor 进行链接的`transform` 和 `fit` 方法。

重要的一点是，开发人员和使用者在实现他们的算法时不需要考虑链接问题，所有的链接问题都会被 FlinkML 自动处理。

### 如何实现一个 Pipeline 操作

为了支持 FlinkML 中的管道操作 (pipelining)，算法必须遵循一个设计模式，我们会在这一章节描述该设计模式。
假设我们希望想要实现一个 pipeline 操作，该操作能改变你所提供的数据的平均值。由于居中数据 (centering data) 在许多分析管道 (pipeline) 中是一个常用的预处理步骤，我们将以一个 `Transformer` 的形式实现它。因此我们首先创建一个 `MeanTransformer` 类，该类继承 `Transformer`。

{% highlight scala %}
class MeanTransformer extends Transformer[MeanTransformer] {}
{% endhighlight %}

因为我们希望能配置作为结果的数据的平均值，所以我们必须加一个配置参数。

{% highlight scala %}
class MeanTransformer extends Transformer[MeanTransformer] {
  def setMean(mean: Double): this.type = {
    parameters.add(MeanTransformer.Mean, mean)
    this
  }
}

object MeanTransformer {
  case object Mean extends Parameter[Double] {
    override val defaultValue: Option[Double] = Some(0.0)
  }

  def apply(): MeanTransformer = new MeanTransformer
}
{% endhighlight %}

参数被定义在 transformer 类的伴生对象中，并且继承了 `Parameter` 类。因为对于参数映射，参数实例应作为不可变的键，因此它以 `case objects` (样本对象) 的形式实现。如果用户没有设置其它的值，默认值就会被使用。如果默认值没有指明，意味着 `defaultValue = None`， 该算法需要根据情况进行处理。

我们现在实例化一个 `MeanTransformer` 对象，并且设置平均值为 t。rmed data。
但是我们仍然需要实现 transformation 是如何工作的。
该工作流可以被分成两个阶段。
在第一个阶段里，transformer 学习给定的训练数据的平均值。
这个知识可以在第二个阶段中被用来转换被提供的且与被配置作为结果的平均值有关的数据。

平均值的学习能够通过我们 `Transformer` 中，从 `Estimator` 继承而来的 `fit` 操作实现。
在 `fit` 操作中，一个与给定的训练数据有关的 pipeline 组件被训练。
然而，该算法**不是**通过覆写 `fit` 方法实现，而是通过提供一个与正确类型对应的一个 `FitOperation` 来实现。
通过查看 `Transformer` 的父类 `Estimator` 的 `fit` 方法的定义，了解为什么是这种情况。

{% highlight scala %}
trait Estimator[Self] extends WithParameters with Serializable {
  that: Self =>

  def fit[Training](
      training: DataSet[Training],
      fitParameters: ParameterMap = ParameterMap.Empty)
      (implicit fitOperation: FitOperation[Self, Training]): Unit = {
    FlinkMLTools.registerFlinkMLTypes(training.getExecutionEnvironment)
    fitOperation.fit(this, fitParameters, training)
  }
}
{% endhighlight %}

我们发现 `fit` 方法调用时需要一个 `Training` 类型的输入集，一个可选参数列表，和第二个带一个 `FitOperation` 类隐式参数的参数列表
在方法体中，首先某个机器学习类型被注册，接着 `FitOperation` 参数的 `fit` 方法被调用。
该实例把自身，参数映射和训练数据集作为一个参数提供给方法。
因此，所有的程序逻辑发生在 `FitOperation`。

`FitOperation` 有两类参数。
第一个定义了需要 `FitOperation` 为之工作的 pipeline 操作类型，而第二个类型参数定义了数据集元素的类型。
如果我们希望先实现 `MeanTransformer`， 使之能工作在 `DenseVector` 上，我们则需要为 `FitOperation[MeanTransformer, DenseVector]` 提供一个实现。

{% highlight scala %}
val denseVectorMeanFitOperation = new FitOperation[MeanTransformer, DenseVector] {
  override def fit(instance: MeanTransformer, fitParameters: ParameterMap, input: DataSet[DenseVector]) : Unit = {
    import org.apache.flink.ml.math.Breeze._
    val meanTrainingData: DataSet[DenseVector] = input
      .map{ x => (x.asBreeze, 1) }
      .reduce{
        (left, right) =>
          (left._1 + right._1, left._2 + right._2)
      }
      .map{ p => (p._1/p._2).fromBreeze }
  }
}
{% endhighlight %}

`FitOperation[T, I]` 有一个 `fit` 方法，该方法在调用时需要一个 `T` 类型的实例，一个参数映射和一个输入 `DataSet[I]`。
在我们的例子中，`T=MeanTransformer` 且 `I=DenseVector`。
如果我们的拟合步骤所需的参数在 `Transformer` 创建时没有给定，则需要参数映射。
`MeanTransformer` 的 `FitOperation` 对给定的输入数据集 `DenseVector` instances 求和，并将结果除以向量的总数。
这样的话，我们获得一个含有平均值的 `DataSet[DenseVector]`。

如果我们再仔细了解这个实现，可以发现平均值计算的结果并没有在任何代码中被储存。
我们想要在后面的步骤中使用这个结果或“知识”来调整我们输入的平均值，我们需要保存它。
而这就是提供给 `fit` 方法的 `MeanTransformer` 类型参数发挥作用的时候。
我们能够使用这个实例来存储状态，该状态能够被接下来工作在同一个对象上的 `transform` 操作所使用。
但是首先，我们必须通过一个成员字段继承 `MeanTransformer` ，并调整 `FitOperation` 的实现。

{% highlight scala %}
class MeanTransformer extends Transformer[Centering] {
  var meanOption: Option[DataSet[DenseVector]] = None

  def setMean(mean: Double): Mean = {
    parameters.add(MeanTransformer.Mean, mu)
  }
}

val denseVectorMeanFitOperation = new FitOperation[MeanTransformer, DenseVector] {
  override def fit(instance: MeanTransformer, fitParameters: ParameterMap, input: DataSet[DenseVector]) : Unit = {
    import org.apache.flink.ml.math.Breeze._

    instance.meanOption = Some(input
      .map{ x => (x.asBreeze, 1) }
      .reduce{
        (left, right) =>
          (left._1 + right._1, left._2 + right._2)
      }
      .map{ p => (p._1/p._2).fromBreeze })
  }
}
{% endhighlight %}

如果我们观察 `Transformer` 中的 `transform` 方法，可以发现我们还需要 `TransformOperation` 的实现。
以下是平均值转换实现的其中一种方式。

{% highlight scala %}

val denseVectorMeanTransformOperation = new TransformOperation[MeanTransformer, DenseVector, DenseVector] {
  override def transform(
      instance: MeanTransformer,
      transformParameters: ParameterMap,
      input: DataSet[DenseVector])
    : DataSet[DenseVector] = {
    val resultingParameters = parameters ++ transformParameters

    val resultingMean = resultingParameters(MeanTransformer.Mean)

    instance.meanOption match {
      case Some(trainingMean) => {
        input.map{ new MeanTransformMapper(resultingMean) }.withBroadcastSet(trainingMean, "trainingMean")
      }
      case None => throw new RuntimeException("MeanTransformer has not been fitted to data.")
    }
  }
}

class MeanTransformMapper(resultingMean: Double) extends RichMapFunction[DenseVector, DenseVector] {
  var trainingMean: DenseVector = null

  override def open(parameters: Configuration): Unit = {
    trainingMean = getRuntimeContext().getBroadcastVariable[DenseVector]("trainingMean").get(0)
  }

  override def map(vector: DenseVector): DenseVector = {
    import org.apache.flink.ml.math.Breeze._

    val result = vector.asBreeze - trainingMean.asBreeze + resultingMean

    result.fromBreeze
  }
}
{% endhighlight %}

现在我们所需的都实现了，可以对一个 `DenseVector` 实例训练数据集拟合我们的 `MeanTransformer` 并 转换他们。
然而，当我们执行 `fit` 操作

{% highlight scala %}
val trainingData: DataSet[DenseVector] = ...
val meanTransformer = MeanTransformer()

meanTransformer.fit(trainingData)
{% endhighlight %}

我们在运行程序时收到以下错误信息: `"There is no FitOperation defined for class MeanTransformer which trains on a DataSet[org.apache.flink.ml.math.DenseVector]"`.
原因是 Scala 编译器不能为 `fit` 方法的隐式参数找到一个符合 `FitOperation` 正确类型的值。
因此，它选择了最原先的隐式值，而该值在运行时引发了这个错误信息。
为了使编译器能理解我们的实现，我们必须讲它作为一个隐式值定义，并将之放在在 `MeanTransformer` 的伴生对象中。

{% highlight scala %}
object MeanTransformer{
  implicit val denseVectorMeanFitOperation = new FitOperation[MeanTransformer, DenseVector] ...

  implicit val denseVectorMeanTransformOperation = new TransformOperation[MeanTransformer, DenseVector, DenseVector] ...
}
{% endhighlight %}

现在我们以 `DataSet[DenseVector]` 为输入调用我们的 `MeanTransformer` 的 `fit` 和 `transform` 方法。
不仅如此，我们现在能把 transformer 作为分析管道的一部分来使用，该管道有一个 `DenseVector` 作为输入和一个输出。

{% highlight scala %}
val trainingData: DataSet[DenseVector] = ...

val mean = MeanTransformer.setMean(1.0)
val polyFeatures = PolynomialFeatures().setDegree(3)

val pipeline = mean.chainTransformer(polyFeatures)

pipeline.fit(trainingData)
{% endhighlight %}

值得注意的是，不需要额外的代码来使用链接
该系统会自动使用每个部件的操作来自动构建 pipeline 逻辑。

到目前为止，用 `DenseVector` 作为输入一切顺利。
但如果我们使用 `LabeledVector` 来调 transformer，会发生什么情况？

{% highlight scala %}
val trainingData: DataSet[LabeledVector] = ...

val mean = MeanTransformer()

mean.fit(trainingData)
{% endhighlight %}

跟之前相同，在运行程序时抛出了以下异常信息: `"There is no FitOperation defined for class MeanTransformer which trains on a DataSet[org.apache.flink.ml.common.LabeledVector]"`.
值得留意的是，这个异常在准备阶段就被抛出，意味着此时工作还没被提交到运行系统。
这有一个很大的好处：你不会看到一个工作在跑了若干天之后由于 pipeline 组件的不兼容而失败。
因此，类型的兼容性在整个工作的早期就被检查。

In order to make the `MeanTransformer` work on `LabeledVector` as well, we have to provide the corresponding operations.
Consequently, we have to define a `FitOperation[MeanTransformer, LabeledVector]` and `TransformOperation[MeanTransformer, LabeledVector, LabeledVector]` as implicit values in the scope of `MeanTransformer`'s companion object.

{% highlight scala %}
object MeanTransformer {
  implicit val labeledVectorFitOperation = new FitOperation[MeanTransformer, LabeledVector] ...

  implicit val labeledVectorTransformOperation = new TransformOperation[MeanTransformer, LabeledVector, LabeledVector] ...
}
{% endhighlight %}

If we wanted to implement a `Predictor` instead of a `Transformer`, then we would have to provide a `FitOperation`, too.
Moreover, a `Predictor` requires a `PredictOperation` which implements how predictions are calculated from testing data.  
