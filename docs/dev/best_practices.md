---
title: "Best Practices"
nav-parent_id: dev
nav-pos: 90
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


本章包含了一系列关于flink 编程人员如何处理常见问题的最佳实践

## 在你的flink应用中解析和传递命令行参数

大多数Flink应用，包含批量计量和流式计算都依赖外部配置参数。

例如 指定输入和输出来源（像路径和地址）、系统参数（并行、运行时配置）和应用参数（经常在用户的函数中使用到）.

从0.9版本我们就提供了一个简单的基本工具：`ParameterTool`，用于解决这些问题。

你也可以不使用这里提到的`ParameterTool`工具。其他框架，例如[Commons CLI](https://commons.apache.org/proper/commons-cli/),
[argparse4j](http://argparse4j.sourceforge.net/) 和flink也可以集成得很好。

### 深入了解`ParameterTool`的配置项

`ParameterTool`提供了一系列预定义好的读取配置项的表态方法。这个工具内部只需要一个`Map<String, String>`参数，所以它非常容易和你的配置风格集成。

#### 了解 `.properties`的文件

下面方法将读取一个[Properties](https://docs.oracle.com/javase/tutorial/essential/environment/properties.html)文件，然后提供键/值对：

{% highlight java %}
String propertiesFile = "/home/sam/flink/myjob.properties";
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFile);
{% endhighlight %}

#### 了解命令行参数

下面例子将从命令行得到`--input hdfs:///mydata --elements 42`参数
{% highlight java %}
public static void main(String[] args) {
	ParameterTool parameter = ParameterTool.fromArgs(args);
	// .. regular code ..
{% endhighlight %}


#### 了解系统属性

当启动jvm时，你可以设置系统属性：`-Dinput=hdfs:///mydata`。你也可以使用如下系统属性初始化`ParameterTool`：

{% highlight java %}
ParameterTool parameter = ParameterTool.fromSystemProperties();
{% endhighlight %}


### 使用Flink应用中的参数

现在我们已经知道如何从多种途径来获取参数（参考上面的例子），

**直接了解`ParameterTool`**

`ParameterTool`本身有获取这些值的方法
{% highlight java %}
ParameterTool parameters = // ...
parameter.getRequired("input");
parameter.get("output", "myDefaultValue");
parameter.getLong("expectedCount", -1L);
parameter.getNumberOfParameters()
// .. there are more methods available.
{% endhighlight %}

你可以在main()方法中直接使用这些方法的返回值(=客户端提交应用).
例如你可以像这样设置使用方的并发数：

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
int parallelism = parameters.get("mapParallelism", 2);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).setParallelism(parallelism);
{% endhighlight %}

因为`ParameterTool`是可序列化的，所以你可以像这样把他传递给函数本身:
{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer(parameters));
{% endhighlight %}

然后在方法内部使用命令行上的值

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer(parameters));
{% endhighlight %}


#### 将一个配置项传递给简单方法

下例将展示如何将参数作为配置对象传递给用户定义的方法。
{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).withParameters(parameters.getConfiguration())
{% endhighlight %}


在`Tokenizer`里，可以通过`open(Configuration conf)`方法来获取这个对象。
{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {
	@Override
	public void open(Configuration parameters) throws Exception {
		parameters.getInteger("myInt", -1);
		// .. do
{% endhighlight %}

#### 注册全局参数

在`ExecutionConfig`把参数注册为全局任务参数，你可以通过任务管理的web接口和用户定义的方法来获取这些配置

**注册全局参数**

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);

//建立执行环境
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(parameters);
{% endhighlight %}

在用户方法中获取这些全局参数

{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

	@Override
	public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
		ParameterTool parameters = (ParameterTool) getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
		parameters.getRequired("input");
		// .. do more ..
{% endhighlight %}

## 声明大TupleX类型

在多字段的数据类型中，强烈推荐使用POJOs(Plain old Java objects)代替`TupleX`。POJOs也可以用来给大`Tuple`类型命名。

**案例**


在使用
~~~java
Tuple11<String, String, ..., String> var = new ...;
~~~

我们更倾向于创建一个扩展了大Tuple类型的自定义类型：
~~~java
CustomType var = new ...;

public static class CustomType extends Tuple11<String, String, ..., String> {
    // constructor matching super
}
~~~

## 使用Logback代替Log4j


**注：本手册适用于Flink 0.10后的版本**

Apache Flink在代码中使用slf4j作日志抽象接口。我们也建议用户在他们的用户方法中使用sfl4j。

Sfl4j是一个可以在运行时使用不同日志实现(例如：[log4j](http://logging.apache.org/log4j/2.x/) 或 [Logback](http://logback.qos.ch/))的编译时日志接口。

Flink 默认依赖Log4j。本章将介绍如何使用Logback。用户也能使用本手册通过Graylog建立中心化的日志。

使用如下代码，获取日志实例：

{% highlight java %}
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyClass implements MapFunction {
	private static final Logger LOG = LoggerFactory.getLogger(MyClass.class);
	// ...
{% endhighlight %}


### 在IDE外或JAVA应用中运行Flink时使用Logback

在通过依赖管理软件如Maven配置类路径执行类的情况下，Flink将把log4j添加到类路径。

因此，你需要把log4j从Flink的依赖中排除掉，下面的配置假设是一个从[Flink quickstart](../quickstart/java_api_quickstart.html)创建出来的Maven项目。
你可以像这样来修改项目的`pom.xml`文件：



{% highlight xml %}
<dependencies>
	<!-- Add the two required logback dependencies -->
	<dependency>
		<groupId>ch.qos.logback</groupId>
		<artifactId>logback-core</artifactId>
		<version>1.1.3</version>
	</dependency>
	<dependency>
		<groupId>ch.qos.logback</groupId>
		<artifactId>logback-classic</artifactId>
		<version>1.1.3</version>
	</dependency>

	<!-- Add the log4j -> sfl4j (-> logback) bridge into the classpath
	 Hadoop is logging to log4j! -->
	<dependency>
		<groupId>org.slf4j</groupId>
		<artifactId>log4j-over-slf4j</artifactId>
		<version>1.7.7</version>
	</dependency>

	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-java</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-streaming-java{{ site.scala_version_suffix }}</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>org.apache.flink</groupId>
		<artifactId>flink-clients{{ site.scala_version_suffix }}</artifactId>
		<version>{{ site.version }}</version>
		<exclusions>
			<exclusion>
				<groupId>log4j</groupId>
				<artifactId>*</artifactId>
			</exclusion>
			<exclusion>
				<groupId>org.slf4j</groupId>
				<artifactId>slf4j-log4j12</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
</dependencies>
{% endhighlight %}


下列`<dependencies>`部份已经修改完成

 从Flink依赖中排除所有`log4j` 的依赖：
 Maven将忽略Flink中log4j的相关依赖

 从Flink依赖中排除`slf4j-log4j12`部份：
 因为我们将绑定slf4j和logback,必须把slf4j和log4j的绑定关系删除掉

 * 添加Logback依赖：`logback-core` 和 `logback-classic`

 * 添加log4j-over-slf4j的依赖。`log4j-over-slf4j`是一个允许应用程序直接使用Log4j接口来使用Slf4j接口的工具。Flink依赖的Hadoop直接使用Log4j来记录日志。所以我们必须重新所有的日志请求从Log4j到Slf4j，然后再记录到Logback。

你必须手动在所有你正在添加到pom文件中的Flink新依赖项中排除掉这些排除项。
你也需要检查下非Flink的其他依赖是否绑定了log4j.你可以通过`mvn dependency:tree`来分析项目中的依赖。



### 在Flink作为一个集群运行时使用Logback

本手册同样适用于以独立集群的方式在YARN上运行Flink

为了在Flink中使用Logback代替Log4j,你需要在`lib/`目录下删除`log4j-1.2.xx.jar` 和 `sfl4j-log4j12-xxx.jar`

然后，你需要在`lib/`目录下添加如下jar文件：

 * `logback-classic.jar`
 * `logback-core.jar`
 * `log4j-over-slf4j.jar`:
* 这个jar需要在类路径下存在，用来将从Hadoop的日志请求（使用Log4j）重定向到Slf4j

请注意，在使用YARN集群时，你需要显式设置`lib/`目录

在YARN上使用Flink,设置自定义的日志命令是：`./bin/flink run -yt $FLINK_HOME/lib <... remaining arguments ...>`
