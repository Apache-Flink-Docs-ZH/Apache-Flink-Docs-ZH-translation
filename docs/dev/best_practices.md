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

This page contains a collection of best practices for Flink programmers on how to solve frequently encountered problems.

本章包含了一系列关于flink 编程人员如何处理常见问题的最佳实践

* This will be replaced by the TOC
{:toc}

## Parsing command line arguments and passing them around in your Flink application

## 在你的flink应用中解析和传递命令行参数


Almost all Flink applications, both batch and streaming rely on external configuration parameters.
For example for specifying input and output sources (like paths or addresses), also system parameters (parallelism, runtime configuration) and application specific parameters (often used within the user functions).

大多数Flink应用，包含批量计量和流式计算都依赖外部配置参数。

例如 指定输入和输出来源（像路径和地址）、系统参数（并行、运行时配置）和应用参数（经常在用户的函数中使用到）.

Since version 0.9 we are providing a simple utility called `ParameterTool` to provide at least some basic tooling for solving these problems.

从0.9版本我们就提供了一个简单的基本工具：`ParameterTool`，用于解决这些问题。

Please note that you don't have to use the `ParameterTool` explained here. Other frameworks such as [Commons CLI](https://commons.apache.org/proper/commons-cli/),
[argparse4j](http://argparse4j.sourceforge.net/) and others work well with Flink as well.

你也可以不使用这里提到的`ParameterTool`工具。其他框架，例如[Commons CLI](https://commons.apache.org/proper/commons-cli/),
[argparse4j](http://argparse4j.sourceforge.net/) 和flink也可以集成得很好。


### Getting your configuration values into the `ParameterTool`

### 深入了解`ParameterTool`的配置项

The `ParameterTool` provides a set of predefined static methods for reading the configuration. The tool is internally expecting a `Map<String, String>`, so its very easy to integrate it with your own configuration style.

`ParameterTool`提供了一系列预定义好的读取配置项的表态方法。这个工具内部只需要一个`Map<String, String>`参数，所以它非常容易和你的配置风格集成。


#### From `.properties` files
#### 了解 `.properties`的文件

The following method will read a [Properties](https://docs.oracle.com/javase/tutorial/essential/environment/properties.html) file and provide the key/value pairs:
下面方法将读取一个[Properties](https://docs.oracle.com/javase/tutorial/essential/environment/properties.html)文件，然后提供键/值对：

{% highlight java %}
String propertiesFile = "/home/sam/flink/myjob.properties";
ParameterTool parameter = ParameterTool.fromPropertiesFile(propertiesFile);
{% endhighlight %}


#### From the command line arguments
#### 了解命令行参数


This allows getting arguments like `--input hdfs:///mydata --elements 42` from the command line.
下面例子将从命令行得到`--input hdfs:///mydata --elements 42`参数
{% highlight java %}
public static void main(String[] args) {
	ParameterTool parameter = ParameterTool.fromArgs(args);
	// .. regular code ..
{% endhighlight %}



#### From system properties
#### 了解系统属性

When starting a JVM, you can pass system properties to it: `-Dinput=hdfs:///mydata`. You can also initialize the `ParameterTool` from these system properties:
当启动jvm时，你可以设置系统属性：`-Dinput=hdfs:///mydata`。你也可以使用如下系统属性初始化`ParameterTool`：

{% highlight java %}
ParameterTool parameter = ParameterTool.fromSystemProperties();
{% endhighlight %}




### Using the parameters in your Flink program
### 使用Flink应用中的参数

Now that we've got the parameters from somewhere (see above) we can use them in various ways.

现在我们已经知道如何从多种途径来获取参数（参考上面的例子），

**Directly from the `ParameterTool`**
**直接了解`ParameterTool`**

The `ParameterTool` itself has methods for accessing the values.
`ParameterTool`本身有获取这些值的方法
{% highlight java %}
ParameterTool parameters = // ...
parameter.getRequired("input");
parameter.get("output", "myDefaultValue");
parameter.getLong("expectedCount", -1L);
parameter.getNumberOfParameters()
// .. there are more methods available.
{% endhighlight %}


You can use the return values of these methods directly in the main() method (=the client submitting the application).
For example you could set the parallelism of a operator like this:
你可以在main()方法中直接使用这些方法的返回值(=客户端提交应用).
例如你可以像这样设置使用方的并发数：

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
int parallelism = parameters.get("mapParallelism", 2);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).setParallelism(parallelism);
{% endhighlight %}


Since the `ParameterTool` is serializable, you can pass it to the functions itself:
因为`ParameterTool`是可序列化的，所以你可以像这样把他传递给函数本身:
{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer(parameters));
{% endhighlight %}

and then use them inside the function for getting values from the command line.

然后在方法内部使用命令行上的值

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer(parameters));
{% endhighlight %}



#### Passing it as a `Configuration` object to single functions
#### 将一个配置项传递给简单方法

The example below shows how to pass the parameters as a `Configuration` object to a user defined function.
下例将展示如何将参数作为配置对象传递给用户定义的方法。
{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);
DataSet<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).withParameters(parameters.getConfiguration())
{% endhighlight %}



In the `Tokenizer`, the object is now accessible in the `open(Configuration conf)` method:
在`Tokenizer`里，可以通过`open(Configuration conf)`方法来获取这个对象。
{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {
	@Override
	public void open(Configuration parameters) throws Exception {
		parameters.getInteger("myInt", -1);
		// .. do
{% endhighlight %}


#### Register the parameters globally
#### 注册全局参数

Parameters registered as a global job parameter at the `ExecutionConfig` allow you to access the configuration values from the JobManager web interface and all functions defined by the user.

在`ExecutionConfig`把参数注册为全局任务参数，你可以通过任务管理的web接口和用户定义的方法来获取这些配置

**Register the parameters globally**
**注册全局参数**

{% highlight java %}
ParameterTool parameters = ParameterTool.fromArgs(args);

// set up the execution environment
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(parameters);
{% endhighlight %}

Access them in any rich user function:
在用户方法中获取这些全局参数

{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

	@Override
	public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
		ParameterTool parameters = (ParameterTool) getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
		parameters.getRequired("input");
		// .. do more ..
{% endhighlight %}


## Naming large TupleX types
## 声明大TupleX类型

It is recommended to use POJOs (Plain old Java objects) instead of `TupleX` for data types with many fields.
Also, POJOs can be used to give large `Tuple`-types a name.
在多字段的数据类型中，强烈推荐使用POJOs(Plain old Java objects)代替`TupleX`。POJOs也可以用来给大`Tuple`类型命名。

**Example**
**案例**

Instead of using:


~~~java
Tuple11<String, String, ..., String> var = new ...;
~~~


It is much easier to create a custom type extending from the large Tuple type.

~~~java
CustomType var = new ...;

public static class CustomType extends Tuple11<String, String, ..., String> {
    // constructor matching super
}
~~~

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

## Using Logback instead of Log4j
## 使用Logback代替Log4j

**Note: This tutorial is applicable starting from Flink 0.10**
**注：本手册适用于Flink 0.10后的版本**

Apache Flink is using [slf4j](http://www.slf4j.org/) as the logging abstraction in the code. Users are advised to use sfl4j。 as well in their user functions.
Apache Flink在代码中使用slf4j作日志抽象接口。我们也建议用户在他们的用户方法中使用sfl4j。

Sfl4j is a compile-time logging interface that can use different logging implementations at runtime, such as [log4j](http://logging.apache.org/log4j/2.x/) or [Logback](http://logback.qos.ch/).
Sfl4j是一个可以在运行时使用不同日志实现(例如：[log4j](http://logging.apache.org/log4j/2.x/) 或 [Logback](http://logback.qos.ch/))的编译时日志接口。

Flink is depending on Log4j by default. This page describes how to use Flink with Logback. Users reported that they were also able to set up centralized logging with Graylog using this tutorial.

Flink 默认依赖Log4j。本章将介绍如何使用Logback。用户也能使用本手册通过Graylog建立中心化的日志。

To get a logger instance in the code, use the following code:
使用如下代码，获取日志实例：

{% highlight java %}
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyClass implements MapFunction {
	private static final Logger LOG = LoggerFactory.getLogger(MyClass.class);
	// ...
{% endhighlight %}


### Use Logback when running Flink out of the IDE / from a Java application
### 在IDE外或JAVA应用中运行Flink时使用Logback

In all cases were classes are executed with a classpath created by a dependency manager such as Maven, Flink will pull log4j into the classpath.
在通过依赖管理软件如Maven配置类路径执行类的情况下，Flink将把log4j添加到类路径。

Therefore, you will need to exclude log4j from Flink's dependencies. The following description will assume a Maven project created from a [Flink quickstart](../quickstart/java_api_quickstart.html).

Change your projects `pom.xml` file like this:

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

The following changes were done in the `<dependencies>` section:
下列`<dependencies>`部份已经修改完成

 * Exclude all `log4j` dependencies from all Flink dependencies: This causes Maven to ignore Flink's transitive dependencies to log4j.
 从Flink依赖中排除所有`log4j` 的依赖：Maven将忽略Flink中log4j的相关依赖

 * Exclude the `slf4j-log4j12` artifact from Flink's dependencies: Since we are going to use the slf4j to logback binding, we have to remove the slf4j to log4j binding.
 从Flink依赖中排除`slf4j-log4j12`部份：
 因为我们将绑定slf4j和logback,必须把slf4j和log4j的绑定关系删除掉

 * Add the Logback dependencies: `logback-core` and `logback-classic`
 * 添加Logback依赖：`logback-core` 和 `logback-classic`
 * Add dependencies for `log4j-over-slf4j`. `log4j-over-slf4j` is a tool which allows legacy applications which are directly using the Log4j APIs to use the Slf4j interface. Flink depends on Hadoop which is directly using Log4j for logging. Therefore, we need to redirect all logger calls from Log4j to Slf4j which is in turn logging to Logback.
 * 添加log4j-over-slf4j的依赖。`log4j-over-slf4j`是一个允许应用程序直接使用Log4j接口来使用Slf4j接口的工具。Flink依赖的Hadoop直接使用Log4j来记录日志。所以我们必须重新所有的日志请求从Log4j到Slf4j，然后再记录到Logback。

Please note that you need to manually add the exclusions to all new Flink dependencies you are adding to the pom file.
你必须手动在所有你正在添加到pom文件中的Flink新依赖项中排除掉这些排除项。

You may also need to check if other dependencies (non Flink) are pulling in log4j bindings. You can analyze the dependencies of your project with `mvn dependency:tree`.
你也需要检查下非Flink的其他依赖是否绑定了log4j.你可以通过`mvn dependency:tree`来分析项目中的依赖。



### Use Logback when running Flink on a cluster
### 在Flink作为一个集群运行时使用Logback

This tutorial is applicable when running Flink on YARN or as a standalone cluster.
本手册同样适用于以独立集群的方式在YARN上运行Flink

In order to use Logback instead of Log4j with Flink, you need to remove the `log4j-1.2.xx.jar` and `sfl4j-log4j12-xxx.jar` from the `lib/` directory.
为了在Flink中使用Logback代替Log4j,你需要在`lib/`目录下删除`log4j-1.2.xx.jar` 和 `sfl4j-log4j12-xxx.jar`

Next, you need to put the following jar files into the `lib/` folder:
然后，你需要在`lib/`目录下添加如下jar文件：

 * `logback-classic.jar`
 * `logback-core.jar`
 * `log4j-over-slf4j.jar`: This bridge needs to be present in the classpath for redirecting logging calls from Hadoop (which is using Log4j) to Slf4j.
* 这个jar需要在类路径下存在，用来将从Hadoop的日志请求（使用Log4j）重定向到Slf4j
Note that you need to explicitly set the `lib/` directory when using a per job YARN cluster.
请注意，在使用YARN集群时，你需要显式设置`lib/`目录

The command to submit Flink on YARN with a custom logger is: `./bin/flink run -yt $FLINK_HOME/lib <... remaining arguments ...>`
在YARN上使用Flink,设置自定义的日志命令是：`./bin/flink run -yt $FLINK_HOME/lib <... remaining arguments ...>`
