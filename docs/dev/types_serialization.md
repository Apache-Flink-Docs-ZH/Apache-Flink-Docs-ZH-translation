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
 
* **注册子类型：** 如果函数签名只在父类型中申明，但实际执行中使用的是这些类型的子类型，让Fink知道这些子类型能够提升不少性能。因此，在`StreamExecutionEnvironment`或`ExecutionEnvironment`中应当为每个子类型调用`.registerType(clazz)`方法。

* **注册自定义序列化构造器：** Flink会将自己不能处理的类型转交给[Kryo](https://github.com/EsotericSoftware/kryo),但并不是所有的类型都能被Kryo完美处理（也就是说：不是所有类型都能被Flink处理），比如，许多 Google Guava 的集合类型默认情况下是不能正常工作的。针对存在这个问题的这些类型，其解决方案
是通过在`StreamExecutionEnvironment` 或 `ExecutionEnvironment`中调用`.getConfig().addDefaultKryoSerializer(clazz, serializer)`方法，注册辅助的序列化构造器。许多库都提供了Kryo序列化构造器，关于自定义序列化构造器的更多细节请参考[Custom Serializers]({{ site.baseurl }}/dev/custom_serializers.html)。
* **添加Type Hints：** 有时，Flink尝试了各种办法仍不能推断出泛型，这时用户就必须通过添加`type hint`来推断泛型，一般这种情况只是在Java API中需要添加的。[Type Hints Section(#type-hints-in-the-java-api) 中讲解更为详细。
 
 * **手动创建一个 `TypeInformation`类：** 在某些API中，手动创建一个`TypeInformation`类可能是必须的，因为Java泛型类型的擦除特性会使得Flink无法推断数据类型。更多详细信息可以参考[Creating a TypeInformation or TypeSerializer](#creating-a-typeinformation-or-typeserializer)
 


## FLink的TypeInformation类
 {% gh_link /flink-core/src/main/java/org/apache/flink/api/common/typeinfo/TypeInformation.java "TypeInformation" %}类是所有类型描述类的基类。它包括了类型的一些基本属性，并可以通过它来生成序列化构造器（serializer），特殊情况下还可以生成类型的比较器。（*注意：Flink中的比较器不仅仅是定义大小顺序，更是处理keys的基本辅助工具*）
 在FLink内部，类型有以下区别：
 
 1.    基本类型：所有Java基本数据类型和对应装箱类型，加上void, String, Date
2.    基本数组和Object数组
3.    复合类型：
a.     Flink Java Tuple（Flink Java API的一部分）
b.    Scala case 类（包括Scala Tuple）
c.     POJO类：遵循类bean模式的类
4.    Scala辅助类型（Option，Either，Lists，Maps…）
5.    泛型（Generic）：这些类型将不会由Flink自己序列化，而是借助Kryo来序列化

Internally, Flink makes the following distinctions between types:

* 基本类型：所有Java基本数据类型和对应装箱类型，加上`void`, `String`, `Date`, `BigDecimal`和 `BigInteger`.

* 基本数组和对象数组

* 复合类型：

  * Flink Java Tuples (Flink Java API的一部分): 最多25个成员，不支持null成员

  * Scala *case 类* (包括 Scala tuples): 最多25个成员, 不支持null成员

  * Row: 包含任意多个字段的元组并且支持null成员

  * POJOs: 参照类bean模式的类

* 辅助类型  (Option, Either, Lists, Maps, ...)

* 泛型: Flink自身不会序列化泛型，而是借助Kryo进行序列化.

POJO类非常有意思，因为POJO类可以支持复杂类型的创建，并且在定义keys时可以使用成员的名字：`dataSet.join(another).where("name").equalTo("personName")`。同时，POJO类对于运行时（runtime）是透明的，这使得Flink可以非常高效地处理它们。
  

#### POJO类型的规则
在满足如下条件时，Flink会将这种数据类型识别成POJO类型（并允许以成员名引用）：

* 该类是public的并且是独立的（即没有非静态的内部类）
* 该类有一个public的无参构造函数
* 该类（及该类的父类）的所有成员要么是public的，要么是拥有按照标准java bean命名规则命名的public getter和 public setter方法。
* 

* The class is public and standalone (no non-static inner class)
* The class has a public no-argument constructor
* All fields in the class (and all superclasses) are either public (and non-final)
  or have a public getter- and a setter- method that follows the Java beans
  naming conventions for getters and setters.


#### Creating a TypeInformation or TypeSerializer

To create a TypeInformation object for a type, use the language specific way:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
Because Java generally erases generic type information, you need to pass the type to the TypeInformation
construction:

For non-generic types, you can pass the Class:
{% highlight java %}
TypeInformation<String> info = TypeInformation.of(String.class);
{% endhighlight %}

For generic types, you need to "capture" the generic type information via the `TypeHint`:
{% highlight java %}
TypeInformation<Tuple2<String, Double>> info = TypeInformation.of(new TypeHint<Tuple2<String, Double>>(){});
{% endhighlight %}
Internally, this creates an anonymous subclass of the TypeHint that captures the generic information to preserve it
until runtime.
</div>

<div data-lang="scala" markdown="1">
In Scala, Flink uses *macros* that runs at compile time and captures all generic type information while it is
still available.
{% highlight scala %}
// important: this import is needed to access the 'createTypeInformation' macro function
import org.apache.flink.streaming.api.scala._

val stringInfo: TypeInformation[String] = createTypeInformation[String]

val tupleInfo: TypeInformation[(String, Double)] = createTypeInformation[(String, Double)]
{% endhighlight %}

You can still use the same method as in Java as a fallback.
</div>
</div>

To create a `TypeSerializer`, simply call `typeInfo.createSerializer(config)` on the `TypeInformation` object.

The `config` parameter is of type `ExecutionConfig` and holds the information about the program's registered
custom serializers. Where ever possibly, try to pass the programs proper ExecutionConfig. You can usually
obtain it from `DataStream` or `DataSet` via calling `getExecutionConfig()`. Inside functions (like `MapFunction`), you can
get it by making the function a [Rich Function]() and calling `getRuntimeContext().getExecutionConfig()`.

--------
--------

## Type Information in the Scala API

Scala has very elaborate concepts for runtime type information though *type manifests* and *class tags*. In
general, types and methods have access to the types of their generic parameters - thus, Scala programs do
not suffer from type erasure as Java programs do.

In addition, Scala allows to run custom code in the Scala Compiler through Scala Macros - that means that some Flink
code gets executed whenever you compile a Scala program written against Flink's Scala API.

We use the Macros to look at the parameter types and return types of all user functions during compilation - that
is the point in time when certainly all type information is perfectly available. Within the macro, we create
a *TypeInformation* for the function's return types (or parameter types) and make it part of the operation.


#### No Implicit Value for Evidence Parameter Error

In the case where TypeInformation could not be created, programs fail to compile with an error
stating *"could not find implicit value for evidence parameter of type TypeInformation"*.

A frequent reason if that the code that generates the TypeInformation has not been imported.
Make sure to import the entire flink.api.scala package.
{% highlight scala %}
import org.apache.flink.api.scala._
{% endhighlight %}

Another common cause are generic methods, which can be fixed as described in the following section.


#### Generic Methods

Consider the following case below:

{% highlight scala %}
def selectFirst[T](input: DataSet[(T, _)]) : DataSet[T] = {
  input.map { v => v._1 }
}

val data : DataSet[(String, Long) = ...

val result = selectFirst(data)
{% endhighlight %}

For such generic methods, the data types of the function parameters and return type may not be the same
for every call and are not known at the site where the method is defined. The code above will result
in an error that not enough implicit evidence is available.

In such cases, the type information has to be generated at the invocation site and passed to the
method. Scala offers *implicit parameters* for that.

The following code tells Scala to bring a type information for *T* into the function. The type
information will then be generated at the sites where the method is invoked, rather than where the
method is defined.

{% highlight scala %}
def selectFirst[T : TypeInformation](input: DataSet[(T, _)]) : DataSet[T] = {
  input.map { v => v._1 }
}
{% endhighlight %}


--------
--------


## Type Information in the Java API

In the general case, Java erases generic type information. Flink tries to reconstruct as much type information
as possible via reflection, using the few bits that Java preserves (mainly function signatures and subclass information).
This logic also contains some simple type inference for cases where the return type of a function depends on its input type:

{% highlight java %}
public class AppendOne<T> extends MapFunction<T, Tuple2<T, Long>> {

    public Tuple2<T, Long> map(T value) {
        return new Tuple2<T, Long>(value, 1L);
    }
}
{% endhighlight %}

There are cases where Flink cannot reconstruct all generic type information. In that case, a user has to help out via *type hints*.


#### Type Hints in the Java API

In cases where Flink cannot reconstruct the erased generic type information, the Java API
offers so called *type hints*. The type hints tell the system the type of
the data stream or data set produced by a function:

{% highlight java %}
DataSet<SomeType> result = dataSet
    .map(new MyGenericNonInferrableFunction<Long, SomeType>())
        .returns(SomeType.class);
{% endhighlight %}

The `returns` statement specifies the produced type, in this case via a class. The hints support
type definition via

* Classes, for non-parameterized types (no generics)
* TypeHints in the form of `returns(new TypeHint<Tuple2<Integer, SomeType>>(){})`. The `TypeHint` class
  can capture generic type information and preserve it for the runtime (via an anonymous subclass).


#### Type extraction for Java 8 lambdas

Type extraction for Java 8 lambdas works differently than for non-lambdas, because lambdas are not associated
with an implementing class that extends the function interface.

Currently, Flink tries to figure out which method implements the lambda and uses Java's generic signatures to
determine the parameter types and the return type. However, these signatures are not generated for lambdas
by all compilers (as of writing this document only reliably by the Eclipse JDT compiler from 4.5 onwards).


#### Serialization of POJO types

The PojoTypeInformation is creating serializers for all the fields inside the POJO. Standard types such as
int, long, String etc. are handled by serializers we ship with Flink.
For all other types, we fall back to Kryo.

If Kryo is not able to handle the type, you can ask the PojoTypeInfo to serialize the POJO using Avro.
To do so, you have to call

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceAvro();
{% endhighlight %}

Note that Flink is automatically serializing POJOs generated by Avro with the Avro serializer.

If you want your **entire** POJO Type to be treated by the Kryo serializer, set

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().enableForceKryo();
{% endhighlight %}

If Kryo is not able to serialize your POJO, you can add a custom serializer to Kryo, using
{% highlight java %}
env.getConfig().addDefaultKryoSerializer(Class<?> type, Class<? extends Serializer<?>> serializerClass)
{% endhighlight %}

There are different variants of these methods available.


## Disabling Kryo Fallback

There are cases when programs may want to explicitly avoid using Kryo as a fallback for generic types. The most
common one is wanting to ensure that all types are efficiently serialized either through Flink's own serializers,
or via user-defined custom serializers.

The setting below will raise an exception whenever a data type is encountered that would go through Kryo:
{% highlight java %}
env.getConfig().disableGenericTypes();
{% endhighlight %}


## Defining Type Information using a Factory

A type information factory allows for plugging-in user-defined type information into the Flink type system.
You have to implement `org.apache.flink.api.common.typeinfo.TypeInfoFactory` to return your custom type information. 
The factory is called during the type extraction phase if the corresponding type has been annotated 
with the `@org.apache.flink.api.common.typeinfo.TypeInfo` annotation. 

Type information factories can be used in both the Java and Scala API.

In a hierarchy of types the closest factory 
will be chosen while traversing upwards, however, a built-in factory has highest precedence. A factory has 
also higher precendence than Flink's built-in types, therefore you should know what you are doing.

The following example shows how to annotate a custom type `MyTuple` and supply custom type information for it using a factory in Java.

The annotated custom type:
{% highlight java %}
@TypeInfo(MyTupleTypeInfoFactory.class)
public class MyTuple<T0, T1> {
  public T0 myfield0;
  public T1 myfield1;
}
{% endhighlight %}

The factory supplying custom type information:
{% highlight java %}
public class MyTupleTypeInfoFactory extends TypeInfoFactory<MyTuple> {

  @Override
  public TypeInformation<MyTuple> createTypeInfo(Type t, Map<String, TypeInformation<?>> genericParameters) {
    return new MyTupleTypeInfo(genericParameters.get("T0"), genericParameters.get("T1"));
  }
}
{% endhighlight %}

The method `createTypeInfo(Type, Map<String, TypeInformation<?>>)` creates type information for the type the factory is targeted for. 
The parameters provide additional information about the type itself as well as the type's generic type parameters if available.

If your type contains generic parameters that might need to be derived from the input type of a Flink function, make sure to also 
implement `org.apache.flink.api.common.typeinfo.TypeInformation#getGenericParameters` for a bidirectional mapping of generic 
parameters to type information.
