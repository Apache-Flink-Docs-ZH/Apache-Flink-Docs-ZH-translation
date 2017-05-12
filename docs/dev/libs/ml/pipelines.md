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

能把 transformer 和 predictor 链接起来对任何机器学习库都是一个非常重要的特性。在 FlinkML 中我们希望在提供一个直观的API的同时，能够充分利用 Scala 语言的能力来为我们的 pipelines 提供类型安全的实现。我们希望能够实现的是让API的使用变得简单轻松，让使用者在编译时（在工作开始运行之前）避免类型错误，并且消除在需要长期运行的工作提交后由于数据转换操作错误引起的失败情形，而这类错误在机器学习 pipeline 中是经常发生的。

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

Chaining is achieved by calling `chainTransformer` or `chainPredictor` on an object
of a class that implements `Transformer`. These methods return a `ChainedTransformer` or
`ChainedPredictor` object respectively. As we mentioned, `ChainedTransformer` objects can be
chained further, while `ChainedPredictor` objects cannot. These classes take care of applying
fit, transform, and predict operations for a pair of successive transformers or
a transformer and a predictor. They also act recursively if the length of the
chain is larger than two, since every `ChainedTransformer` defines a `transform` and `fit`
operation that can be further chained with more transformers or a predictor.

It is important to note that developers and users do not need to worry about chaining when
implementing their algorithms, all this is handled automatically by FlinkML.

### 如何实现一个 Pipeline 操作

In order to support FlinkML's pipelining, algorithms have to adhere to a certain design pattern, which we will describe in this section.
Let's assume that we want to implement a pipeline operator which changes the mean of your data.
Since centering data is a common pre-processing step in many analysis pipelines, we will implement it as a `Transformer`.
Therefore, we first create a `MeanTransformer` class which inherits from `Transformer`

{% highlight scala %}
class MeanTransformer extends Transformer[MeanTransformer] {}
{% endhighlight %}

Since we want to be able to configure the mean of the resulting data, we have to add a configuration parameter.

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

Parameters are defined in the companion object of the transformer class and extend the `Parameter` class.
Since the parameter instances are supposed to act as immutable keys for a parameter map, they should be implemented as `case objects`.
The default value will be used if no other value has been set by the user of this component.
If no default value has been specified, meaning that `defaultValue = None`, then the algorithm has to handle this situation accordingly.

We can now instantiate a `MeanTransformer` object and set the mean value of the transformed data.
But we still have to implement how the transformation works.
The workflow can be separated into two phases.
Within the first phase, the transformer learns the mean of the given training data.
This knowledge can then be used in the second phase to transform the provided data with respect to the configured resulting mean value.

The learning of the mean can be implemented within the `fit` operation of our `Transformer`, which it inherited from `Estimator`.
Within the `fit` operation, a pipeline component is trained with respect to the given training data.
The algorithm is, however, **not** implemented by overriding the `fit` method but by providing an implementation of a corresponding `FitOperation` for the correct type.
Taking a look at the definition of the `fit` method in `Estimator`, which is the parent class of `Transformer`, reveals what why this is the case.

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

We see that the `fit` method is called with an input data set of type `Training`, an optional parameter list and in the second parameter list with an implicit parameter of type `FitOperation`.
Within the body of the function, first some machine learning types are registered and then the `fit` method of the `FitOperation` parameter is called.
The instance gives itself, the parameter map and the training data set as a parameters to the method.
Thus, all the program logic takes place within the `FitOperation`.

The `FitOperation` has two type parameters.
The first defines the pipeline operator type for which this `FitOperation` shall work and the second type parameter defines the type of the data set elements.
If we first wanted to implement the `MeanTransformer` to work on `DenseVector`, we would, thus, have to provide an implementation for `FitOperation[MeanTransformer, DenseVector]`.

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

A `FitOperation[T, I]` has a `fit` method which is called with an instance of type `T`, a parameter map and an input `DataSet[I]`.
In our case `T=MeanTransformer` and `I=DenseVector`.
The parameter map is necessary if our fit step depends on some parameter values which were not given directly at creation time of the `Transformer`.
The `FitOperation` of the `MeanTransformer` sums the `DenseVector` instances of the given input data set up and divides the result by the total number of vectors.
That way, we obtain a `DataSet[DenseVector]` with a single element which is the mean value.

But if we look closely at the implementation, we see that the result of the mean computation is never stored anywhere.
If we want to use this knowledge in a later step to adjust the mean of some other input, we have to keep it around.
And here is where the parameter of type `MeanTransformer` which is given to the `fit` method comes into play.
We can use this instance to store state, which is used by a subsequent `transform` operation which works on the same object.
But first we have to extend `MeanTransformer` by a member field and then adjust the `FitOperation` implementation.

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

If we look at the `transform` method in `Transformer`, we will see that we also need an implementation of `TransformOperation`.
A possible mean transforming implementation could look like the following.

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

Now we have everything implemented to fit our `MeanTransformer` to a training data set of `DenseVector` instances and to transform them.
However, when we execute the `fit` operation

{% highlight scala %}
val trainingData: DataSet[DenseVector] = ...
val meanTransformer = MeanTransformer()

meanTransformer.fit(trainingData)
{% endhighlight %}

we receive the following error at runtime: `"There is no FitOperation defined for class MeanTransformer which trains on a DataSet[org.apache.flink.ml.math.DenseVector]"`.
The reason is that the Scala compiler could not find a fitting `FitOperation` value with the right type parameters for the implicit parameter of the `fit` method.
Therefore, it chose a fallback implicit value which gives you this error message at runtime.
In order to make the compiler aware of our implementation, we have to define it as an implicit value and put it in the scope of the `MeanTransformer's` companion object.

{% highlight scala %}
object MeanTransformer{
  implicit val denseVectorMeanFitOperation = new FitOperation[MeanTransformer, DenseVector] ...

  implicit val denseVectorMeanTransformOperation = new TransformOperation[MeanTransformer, DenseVector, DenseVector] ...
}
{% endhighlight %}

Now we can call `fit` and `transform` of our `MeanTransformer` with `DataSet[DenseVector]` as input.
Furthermore, we can now use this transformer as part of an analysis pipeline where we have a `DenseVector` as input and expected output.

{% highlight scala %}
val trainingData: DataSet[DenseVector] = ...

val mean = MeanTransformer.setMean(1.0)
val polyFeatures = PolynomialFeatures().setDegree(3)

val pipeline = mean.chainTransformer(polyFeatures)

pipeline.fit(trainingData)
{% endhighlight %}

It is noteworthy that there is no additional code needed to enable chaining.
The system automatically constructs the pipeline logic using the operations of the individual components.

So far everything works fine with `DenseVector`.
But what happens, if we call our transformer with `LabeledVector` instead?
{% highlight scala %}
val trainingData: DataSet[LabeledVector] = ...

val mean = MeanTransformer()

mean.fit(trainingData)
{% endhighlight %}

As before we see the following exception upon execution of the program: `"There is no FitOperation defined for class MeanTransformer which trains on a DataSet[org.apache.flink.ml.common.LabeledVector]"`.
It is noteworthy, that this exception is thrown in the pre-flight phase, which means that the job has not been submitted to the runtime system.
This has the advantage that you won't see a job which runs for a couple of days and then fails because of an incompatible pipeline component.
Type compatibility is, thus, checked at the very beginning for the complete job.

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
