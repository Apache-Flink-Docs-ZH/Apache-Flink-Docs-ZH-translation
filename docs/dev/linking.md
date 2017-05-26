---
nav-title: "关联可选模块"
title: "关联不包含于发行版的模块"
nav-parent_id: start
nav-pos: 10
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

二进制发行版的 jar 包包含于`lib`文件夹中，这些 jar 包会自行加入你的
分布式程序的 classpath 中。除了少数例外，几乎所有 Flink 的类都可以在那里找到，
例如流式连接器和一些新加入的模块。
为了运行依赖这些模块的代码，你需要确保模块在运行时是可访问的，为此我们有两点建议：

1. 复制必需的 jar 文件到`lib`文件夹，以提供给你所有的 TaskManagers 。注意，复制之后需要重启你的 TaskManagers 。
2. 或者将这些包打包进你的代码。

推荐使用较新的版本，因为它遵循了 FLink 中的类加载管理器

### 使用Maven打包你的用户代码的依赖包

在使用 maven 时，如果想要打包的依赖不在 Flink 包中，建议使用以下两种方法:

1. maven 的 assembly 插件构建了一个所谓的高级 jar 包（可执行 jar 包），可以包含你的所有依赖项。
 assembly 的配置方法很明了，但是得到的 jar 包可能会变得笨重。
了解更多信息，请查看[maven-assembly-plugin](http://maven.apache.org/plugins/maven-assembly-plugin/usage.html) 。
2. 使用 maven 的 unpack 解包插件把相关依赖解包出来，然后打包进你的代码。

使用较新的方法来捆绑 Kafka 连接器`flink-connector-kafka`,为此
你需要同时从连接器和 Kafka API 本身来添加加类。在你的插件配置中添加如下代码。

~~~xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>2.9</version>
    <executions>
        <execution>
            <id>unpack</id>
            <!-- executed just before the package phase -->
            <phase>prepare-package</phase>
            <goals>
                <goal>unpack</goal>
            </goals>
            <configuration>
                <artifactItems>
                    <!-- For Flink connector classes -->
                    <artifactItem>
                        <groupId>org.apache.flink</groupId>
                        <artifactId>flink-connector-kafka</artifactId>
                        <version>{{ site.version }}</version>
                        <type>jar</type>
                        <overWrite>false</overWrite>
                        <outputDirectory>${project.build.directory}/classes</outputDirectory>
                        <includes>org/apache/flink/**</includes>
                    </artifactItem>
                    <!-- For Kafka API classes -->
                    <artifactItem>
                        <groupId>org.apache.kafka</groupId>
                        <artifactId>kafka_<YOUR_SCALA_VERSION></artifactId>
                        <version><YOUR_KAFKA_VERSION></version>
                        <type>jar</type>
                        <overWrite>false</overWrite>
                        <outputDirectory>${project.build.directory}/classes</outputDirectory>
                        <includes>kafka/**</includes>
                    </artifactItem>
                </artifactItems>
            </configuration>
        </execution>
    </executions>
</plugin>
~~~

现在,如果执行`mvn clean package`，产生的 jar 包将包含所需的依赖。
