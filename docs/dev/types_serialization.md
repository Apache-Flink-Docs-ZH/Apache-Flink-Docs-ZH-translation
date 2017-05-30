---
title: "数据类型和序列化"
nav-id: types
nav-parent_id: dev
nav-show_overview: true
nav-pos: 50
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

Apache Flink 处理数据类型和序列化的方式较特别，包括自带的类型描述，一般类型提取和类型序列化框架。本文档描述这些基本概念并解释背后的相关原理。

* This will be replaced by the TOC
{:toc}

## Flink 中的类型处理

Flink会尝试推断出在分布式计算过程中被交换和存储的数据类型的大量信息，你可以想想就像数据库推断表的模式(schema)一样。在大多数情况下，Flink能够完美地推断出所有必须的信息，这些类型信息使得Flink可以做一些很酷的事情： 

* 使用POJOs类型并通过推断的字段名字(如：`dataSet.keyBy("username")`)完成分组(group)/连接(join)/
聚合(aggregate)操作。这些类型信息使得Flink能够提前检查(如拼写错误和类型兼容性)，避免运行时才发现错误。

* Flink知道的数据类型信息越多，序列化和数据布局方案(data layout scheme) 就越好。这对Flink的内存使用范式（memory usage paradigm）非常重要(memory usage paradigm 用于序列化出入堆中的数据，使得序列化的开销非常低)

* 最后，这些信息可以将用户从考虑序列化框架的选择，以及如何类型注册到这些框架中解脱。


一般而言，数据类型的相关信息是在预处理阶段（pre-flight phase）需要，此时程序刚刚调用了 DataStream 和 DataSet，但是还没调用 execute(), print(), count(), 或 collect()方法.

 

## 常见问题
用户在需要使用Flink的数据类型处理时，最常见问题是:
 
* **注册子类型：** 如果方法签名只在父类型中申明，但实际执行中使用的是这些类型的子类型，让Flink知道这些子类型能够提升不少性能。因此，在`StreamExecutionEnvironment`或`ExecutionEnvironment`中应当为每个子类型调用`.registerType(clazz)`方法。

* **注册自定义序列化器：** Flink会将自己不能处理的类型转交给[Kryo](https://github.com/EsotericSoftware/kryo),但并不是所有的类型都能被Kryo完美处理（也就是说：不是所有类型都能被Flink处理），比如，许多 Google Guava 的集合类型默认情况下是不能正常工作的。针对存在这个问题的这些类型，其解决方案
是通过在`StreamExecutionEnvironment` 或 `ExecutionEnvironment`中调用`.getConfig().addDefaultKryoSerializer(clazz, serializer)`方法，注册辅助的序列化器。许多库都提供了Kryo序列化器，关于自定义序列化器的更多细节请参考[Custom Serializers]({{ site.baseurl }}/dev/custom_serializers.html)。
* **添加Type Hints：** 有时，Flink尝试了各种办法仍不能推断出泛型，这时用户就必须通过借助`type hint`来推断泛型，一般这种情况只是在Java API中需要添加的。[Type Hints Section(#type-hints-in-the-java-api) 中讲解更为详细。
 
 * **手动创建一个 `TypeInformation`类：** 在某些API中，手动创建一个`TypeInformation`类可能是必须的，因为Java泛型的类型擦除特性会使得Flink无法推断数据类型。更多详细信息可以参考[Creating a TypeInformation or TypeSerializer](#creating-a-typeinformation-or-typeserializer)
 


## FLink的TypeInformation类
 {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/typeinfo/TypeInformation.java "TypeInformation" %}类是所有类型描述类的基类。它包括了类型的一些基本属性，并可以通过它来生成序列化器（serializer），特殊情况下还可以生成类型的比较器。（*注意：Flink中的比较器不仅仅是定义大小顺序，更是处理keys的基本辅助工具*）
 
* 基本类型：所有Java基本数据类型和对应装箱类型，加上`void`, `String`, `Date`, `BigDecimal`和 `BigInteger`.

* 基本数组和对象数组

* 复合类型：

  * Flink Java Tuples (Flink Java API的一部分): 最多25个成员，不支持null成员

  * Scala *case 类* (包括 Scala tuples): 最多25个成员, 不支持null成员

  * Row: 包含任意多个字段的元组并且支持null成员

  * POJOs: 参照类bean模式的类方法

* 辅助类型  (Option, Either, Lists, Maps, ...)

* 泛型: Flink自身不会序列化泛型，而是借助Kryo进行序列化.

POJO类非常有意思，因为POJO类可以支持复杂类型的创建，并且在定义keys时可以使用成员的名字：`dataSet.join(another).where("name").equalTo("personName")`。同时，POJO类对于运行时（runtime）是透明的，这使得Flink可以非常高效地处理它们。
  

#### POJO类型的规则
在满足如下条件时，Flink会将这种数据类型识别成POJO类型（并允许以成员名引用）：

* 该类是public的并且是独立的（即没有非静态的内部类）
* 该类有一个public的无参构造方法
* 该类（及该类的父类）的所有成员要么是public的，要么是拥有按照标准java bean命名规则命名的public getter和 public setter方法。


#### 创建一个TypeInformation对象或序列化器
#### Creating a TypeInformation or TypeSerializer
创建一个TypeInformation对象时，不同编程语言的创建方法具体如下：
 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
因为Java会对泛型的类型信息进行类型擦除，所以在需要在TypeInformation的构造方法中传入具体类型:
对于非泛型的类型, 你可以直接传入这个类:
{% highlight java %}方法
TypeInformation<String> info = TypeInformation.of(String.class);
{% endhighlight %}

对于泛型类型,你需要借助`TypeHint`去“捕获”泛型的类型信息:
{% highlight java %}
TypeInformation<Tuple2<String, Double>> info = TypeInformation.of(new TypeHint<Tuple2<String, Double>>(){});
{% endhighlight %}
在内部，这个操作创建了一个TypeHint的匿名子类，用于捕获泛型信息，这个子类会一直保存到运行时。
</div>

<div data-lang="scala" markdown="1">
在Scala中，Flink使用在编译时运行的*宏*，在宏可供调用时去捕获所有泛型信息。
In Scala, Flink uses *macros* that runs at compile time and captures all generic type information while it is
still available.
{% highlight scala %}
// 重要: 为了能够访问'createTypeInformation' 的宏方法，这个import是必须的
import org.apache.flink.streaming.api.scala._

val stringInfo: TypeInformation[String] = createTypeInformation[String]

val tupleInfo: TypeInformation[(String, Double)] = createTypeInformation[(String, Double)]
{% endhighlight %}
你也可以在Java使用相同的方法作为备选。
</div>
</div>

为了创建一个`序列化器（TypeSerializer）`，只需要在`TypeInformation` 对象上调用`typeInfo.createSerializer(config)`方法。

`config`参数的类型是`ExecutionConfig`，它保留了程序的注册的自定义序列化器的相关信息。在可能用到TypeSerializer的地方，尽量传入程序的ExecutionConfig，你可以调用`DataStream` 或 `DataSet`的 `getExecutionConfig()`方法获取ExecutionConfig。一些内部方法（如：`Map方法`）中，你可以通过将该方法变成一个[富 方法]()，然后调用`getRuntimeContext().getExecutionConfig()`获取ExecutionConfig.

--------
--------

## Scala API中的类型信息
通过*类型清单(manifests)* and *类标签*功能，Scala对于运行时的类型信息有着非常详细的概念。通常，Scala对象的类型和方法可以访问其泛型参数的类型，因此，Scala程序不会有Java程序那样的类型擦除问题。

此外，Scala允许通过Scala的宏在Scala编译器中运行自定义代码，这意味着当你编译针对Flink的Scala API编写的Scala程序时，会执行一些Flink代码。
在编译期间，我们使用宏来查看所有用户方法的参数类型和返回类型，这也是所有类型信息完全可用的时间点。在宏中，我们为方法的返回类型（或参数类型）创建一个*TypeInformation*，并将其作为宏定义的一部分。 

####  证据参数（Evidence Parameter）缺少隐式值错误
如果无法创建TypeInformation，程序编译报错：“could not find implicit value for evidence parameter of type TypeInformation”。

常见的原因是生成TypeInformation的代码没有被导入，确保导入了完整的flink.api.scala包。
如果生成TypeInformation的代码还没有被导入，这是一个常见的原因 
{% highlight scala %}
import org.apache.flink.api.scala._
{% endhighlight %}

另外一个常见原因是泛型方法，这种可以通过以下部分的内容进行修复。

#### 泛型方法

思考下面这个例子：
{% highlight scala %}
def selectFirst[T](input: DataSet[(T, _)]) : DataSet[T] = {
  input.map { v => v._1 }
}

val data : DataSet[(String, Long) = ...

val result = selectFirst(data)
{% endhighlight %}

例子中的这种泛型方法，方法参数和返回类型的数据类型对每次调用可能不一样，并且定义方法的地方也不知道。这就会导致上面的代码产生一个没有足够的隐式证据可用的错误。
 
在这种情况下，类型信息必须在调用的站点生成并传给方法。Scala为此提供了*隐式参数*。
 
下面的代码告诉Scala把*T*的类型信息带入方法，然后类型信息会在方法被调用的地方生成，而不是在方法定义的位置生成。 

{% highlight scala %}
def selectFirst[T : TypeInformation](input: DataSet[(T, _)]) : DataSet[T] = {
  input.map { v => v._1 }
}
{% endhighlight %}


--------
--------


## Java API中的类型信息
一般情况下，Java会擦除泛型的类型信息。Flink会尝试使用java的保留字节（主要是方法签名和子类信息），通过反射重建尽可能多的类型信息，于方法的返回类型取决于其输入类型的情况，这个逻辑还包含了一些简单的类型推断：
{% highlight java %}
public class AppendOne<T> extends MapFunction<T, Tuple2<T, Long>> {

    public Tuple2<T, Long> map(T value) {
        return new Tuple2<T, Long>(value, 1L);
    }
}
{% endhighlight %}

某些情况下，Flink不能重建所有的类型信息。此时，用户必须通过*type hints*获得帮助。



#### Java API中的类型提示

为了解决Flink无法重建被清除的泛型信息的情况, Java API提供了所谓的*(类型提示)type hint*. 类型提示告诉系统由方法生成的数据流或数据集的类型，如:

{% highlight java %}
DataSet<SomeType> result = dataSet
    .map(new MyGenericNonInferrableFunction<Long, SomeType>())
        .returns(SomeType.class);
{% endhighlight %}

由返回语句指定生成的类型，在本例中是通过一个类。type hint支持以下方式定义类型：
* 类，针对无参数的类型（非泛型）
* 以TypeHints的形式返回`returns(new TypeHint<Tuple2<Integer, SomeType>>(){})`。
 `TypeHint`类可以捕获泛型信息，并将其保留至运行时（通过匿名子类）。



#### 针对Java 8 Lambda表达式的类型抽取
由于Lambda表达式不涉及继承方法接口的实现类，Java 8 Lambda的类型抽取与非lambda表达式的工作方式并不相同。 

当前，Flink正在尝试如何实现Lambda表达式，并使用Java的泛型签名（generic signature）来决定参数类型和返回类型。但是，并不是所有的编译器中的签名都是为了Lambda表达式生成的（本文写作时该文档仅完全适用在Eclipse JDT编译器4.5及以前的编译器下）


#### POJO类型的序列化

PojoTypeInfomation类用于创建针对POJO对象中所有成员的序列化器。Flink自带了针对诸如int，long，String等标准类型的序列化器，对于所有其他的类型，我们交给Kryo处理。

如果Kryo不能处理某种类型，我们可以通过PojoTypeInfo去调用Avro来序列化POJO，为了这样处理，我们必须调用如下接口： 

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceAvro();
{% endhighlight %}

注意：Flink会用Avro序列化器自动序列化由Avro产生POJO对象。

如果你想使得整个POJO类型都交给Kryo序列化器处理，则你需要配置：

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceKryo();
{% endhighlight %}

如果Kryo不能序列化该POJO对象，你可以添加一个自定义序列化器到Kryo，使用代码如下：
{% highlight java %}
env.getConfig().addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass)
{% endhighlight %}

这些方法还有许多不同的重载形式可用


## 禁用 Kryo 作为备选

在某些情况下，程序可能希望明确地避免使用Kryo作为泛型类型的后备。 最常见的情况是想确保所有类型都可以通过Flink自己的序列化器或通过用户定义的自定义序列器有效序列化。
 
当遇到需要通过Kryo序列化的数据类型时，以下设置会引发异常:
{% highlight java %}
env.getConfig().disableGenericTypes();
{% endhighlight %}


## 使用工厂定义类型信息对象
类型信息工厂允许在Flink类型系统中插入用户定义的类型信息，你需要实现`org.apache.flink.api.common.typeinfo.TypeInfoFactory` 用于返回你自定义类型信息。
如果对应的类型已经使用`@org.apache.flink.api.common.typeinfo.TypeInfo`进行注释，则在类型提取阶段会调用你实现的工厂。
 
类型信息工厂在Java和Scala API中均可使用。

在类型的层次结构中，在向上层（子类往父类）移动的过程中会选择最近的工厂，但是内置的工厂具有最高的层次。一个工厂可以比Flink的内置类型还有更高的层次，因为此你应该知道你在做什么。

下面的例子展示了在Java中如何注释一个自定义类型`MyTuole`，并且通过工厂创建自定义的类型信息对象。

注释自定义类型:
{% highlight java %}
@TypeInfo(MyTupleTypeInfoFactory.class)
public class MyTuple<T0, T1> {
  public T0 myfield0;
  public T1 myfield1;
}
{% endhighlight %}

利用工厂创建自定义类型信息对象:
{% highlight java %}
public class MyTupleTypeInfoFactory extends TypeInfoFactory<MyTuple> {

  @Override
  public TypeInformation<MyTuple> createTypeInfo(Type t, Map<String, TypeInformation<?>> genericParameters) {
    return new MyTupleTypeInfo(genericParameters.get("T0"), genericParameters.get("T1"));
  }
}
{% endhighlight %}

`createTypeInfo(Type, Map<String, TypeInformation<?>>)`方法用于创建该工厂针对类型的类型信息，参数和类型的泛型参数可以提供关于该类型的附加信息（如果有参数）。

如果你的类型包含的泛型参数可能源自Flink方法的输入类型，请确保你实现了`org.apache.flink.api.common.typeinfo.TypeInformation#getGenericParameters` 
用于建立从泛型参数到类型信息的双向映射。
