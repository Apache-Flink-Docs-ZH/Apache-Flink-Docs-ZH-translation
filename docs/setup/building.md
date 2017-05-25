---
title: Building Flink from Source
nav-parent_id: start
nav-pos: 20
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

本文主要介绍如何从源码构建Flink {{ site.version }}。

* This will be replaced by the TOC
{:toc}

## 构建Flink
构建 Flink 之前你需要获得源代码。 [下载发布版本源代码]({{ site.download_url }}) 或者 [克隆 git 版本库]({{ site.github_url }}) 均可。

另外你需要 **Maven 3** 和 **JDK**。 构建 Flink JDK版本要求**至少 Java 7**, 我们推荐使用 Java 8。

*提示：Maven 3.3.x 可以用来构建Flink, 但不会很好地消除一些依赖冲突，而Maven 3.0.3 可以正确地创建这些依赖的包。使用 Java 8 构建unit test, 要使用 Java 8u51 或以上的JDK，以避免在使用 PowerMock runner 时unit test 出错。*

从git版本库克隆源码：

~~~bash
git clone {{ site.github_url }}
~~~

以最简单的方式构建 Flink 只需运行：

~~~bash
mvn clean install -DskipTests
~~~

这条命令的意思是 [Maven](http://maven.apache.org) (`mvn`)首先移除所有已经存在的构建(`clean`)并且创建一个新的 Flink 二进制发布版本(`install`)。`-DskipTests`参数禁止Maven执行测试程序。

以默认方式构建源码将包含 Hadoop 2 YARN Client

## 依赖消除

Flink [消除](https://maven.apache.org/plugins/maven-shade-plugin/) 了部分依赖包，以避免与用户自身程序的依赖包出现版本冲突。这部分依赖包有：*Google Guava*, *Asm*, *Apache Curator*, *Apache HTTP Components*，以及其他。

依赖消除机制在最近的 Maven 版本中有所变化， 这就要求用户依据自己的 Maven 版本在构建 Flink 时操作上略有不同。

**Maven 3.0.x, 3.1.x, and 3.2.x**
直接在 Flink 源码包根目录下运行 `mvn clean install -DskipTests` 即可。

**Maven 3.3.x**
构建过程必须分两步：首先在源码根目录构建， 然后再进入 flink-dist 目录构建：

~~~bash
mvn clean install -DskipTests
cd flink-dist
mvn clean install
~~~

*提示:* 查看 Maven 版本, 运行 `mvn --version`。

{% top %}

## Hadoop 版本

大多数用户并不需要手动执行此操作。 [下载页面](http://hadoop.apache.org) 包含常见 Hadoop 版本对应的 Flink 二进制包。

Flink 所依赖的 HDFS 和 YARN 均来自于 [Apache Hadoop](http://hadoop.apache.org)。目前存在多个不同Hadoop版本（包括上游项目及不同Hadoop发行版）。如果使用错误的版本组合，可能会导致异常。

只支持 2.3.0 以上版本的 Hadoop。 你也可以在构建时指定具体的 Hadoop 版本。

~~~bash
mvn clean install -DskipTests -Dhadoop.version=2.6.1
~~~

#### Hadoop 2.3.0 之前版本
Hadoop 2.X 版本仅在 2.3.0 之后支持 YARN 特性。如果需要使用低于 2.3.0 的版本，你可以使用 `-P!include-yarn` 参数移除对于 YARN 的支持。

使用下列命令将会使用Hadoop 2.2.0版本进行构建：

~~~bash
mvn clean install -Dhadoop.version=2.2.0 -P!include-yarn
~~~

### 发行商版本

使用下列命令指定一个发行商版本：

~~~bash
mvn clean install -DskipTests -Pvendor-repos -Dhadoop.version=2.6.1-cdh5.0.0
~~~

使用 `-Pvendor-repos` 表示启动了包含Cloudera, Hortonworks 和 MapR这些当前流行的 Hadoop 发行版本的 Maven  [build profile](http://maven.apache.org/guides/introduction/introduction-to-profiles.html)。

{% top %}

## Scala 版本

{% info %} 用户如果仅使用Java API则可以 *忽略* 这部分内容。

Flink 有一套 [Scala](http://scala-lang.org) 编写的API，代码库和运行时模块。用户在使用Scala API和代码库时需要和自己工程中的 Scala 版本相匹配（因为 Scala 并不是严格意义上的向下兼容）。

**默认情况下, Flink 使用 Scala 2.10 版本进行构建**。你可以使用如下脚本更改默认 Scala *二进制版本*，用来基于 Scala *2.11* 构建 Flink：


~~~bash
# 从 Scala 2.10 版本 切换到 Scala 2.11 版本
tools/change-scala-version.sh 2.11
# 基于 Scala 2.11 版本构建Flink
mvn clean install -DskipTests
~~~

为了根据特定 Scala 版本进行构建，需要切换到相应二进制版本并添加 *语言版本*  作为附加构建属性。 例如，使用 Scala 2.11.4 版本进行构建需要执行：

~~~bash
# 切换到 Scala 2.11 版本
tools/change-scala-version.sh 2.11
# 使用 Scala 2.11.4 版本进行构建
mvn clean install -DskipTests -Dscala.version=2.11.4
~~~

Flink 基于 Scala 2.10 版本开发，另外还经过 Scala 2.11 版本测试。这两个版本是支持的。更早的版本 (如 Scala *2.9*) *不再* 支持.

是否兼容 Scala 的新版本，取决于 Flink 所使用的语言特性是否有重大改变, 以及 Flink 所依赖的组件在新版本 Scala 是否可以获取。 由 Scala 编写的依赖库包括*Kafka*, *Akka*, *Scalatest*, 和 *scopt*。

{% top %}

## 加密文件系统
如果你的 home 目录是加密的，则可能会遇到 `java.io.IOException: File name too long` 异常. 一些加密文件系统，比如 Ubuntu 所使用的 encfs，不允许长文件名，会导致这种错误。

修改办法是添加如下配置:


~~~xml
<args>
    <arg>-Xmax-classfile-name</arg>
    <arg>128</arg>
</args>
~~~

进入导致这个错误的模块的 `pom.xml` 文件的编译配置中。如果错误出现在 `flink-yarn` 模块中，上面的配置需要加入到 `scala-maven-plugin` 中`<configuration>` 标签下。更多信息见 [此问题](https://issues.apache.org/jira/browse/FLINK-2003)。

{% top %}
