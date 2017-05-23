---
title:  "连接器"
nav-parent_id: batch
nav-pos: 4
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

* TOC
{:toc}

## 从文件系统中读取

Flink 内置支持以下文件系统:

| 文件系统                                      |    前缀       |   说明                                   |
| -------------------------------------------- | -------------| --------------------------------------- |
| Hadoop Distributed File System (HDFS) &nbsp; | `hdfs://`    | 支持所有的HDFS版本                         |
| Amazon S3                                    | `s3://`      | 支持实现了Hadoop文件系统的版本(详情见下文)    |
| MapR file system                             | `maprfs://`  | 用户需要手动的将需要的jar包放置在 `lib/` 目录下|
| Alluxio                                      | `alluxio://` | 支持实现了Hadoop文件系统的版本(详情见下文)     |



### 使用实现了Hadoop文件系统的文件系统

Apache Flink 允许用户使用任何实现了`org.apache.hadoop.fs.FileSystem`
接口的文件系统. 下面是实现了Hadoop `FileSystem`的文件系统

- [S3](https://aws.amazon.com/s3/) (已验证)
- [Google Cloud Storage Connector for Hadoop](https://cloud.google.com/hadoop/google-cloud-storage-connector) (已验证)
- [Alluxio](http://alluxio.org/) (已验证)
- [XtreemFS](http://www.xtreemfs.org/) (已验证)
- FTP via [Hftp](http://hadoop.apache.org/docs/r1.2.1/hftp.html) (未验证)
- 其他.

为了在Hadoop文件系统中使用Flink, 必须确保下面几点

- 在Hadoop配置目录中,`flink-conf.yaml`设置了`fs.hdfs.hadoopconf`属性.
- 文件系统需要一个入口文件`core-site.xml`. S3和Alluxio的例子如下.
- 在Flink(所有运行Flink的机器)的安装目录`lib/`下,有系统依赖的class文件.Flink也遵守`HADOOP_CLASSPATH`环境变量,可以将Hadoop jar文件放入路径中,如果不遵从规则,文件将会不可用,.

#### Amazon S3

访问 [Deployment & Operations - Deployment - AWS - S3: Simple Storage Service]({{ site.baseurl }}/setup/aws.html) 获得可用的S3文件系统实现, 他们的配置和需要的类库.

#### Alluxio

将如下配置添加到`core-site.xml`中以支持Alluxio:

~~~xml
<property>
  <name>fs.alluxio.impl</name>
  <value>alluxio.hadoop.FileSystem</value>
</property>
~~~


## 为Hadoop使用 Input/Output格式化 包装器,以连接到其他系统

Apache Flink允许用户接受很多不同的系统作为数据源或者通道.
系统设计得非常容易扩展. 类似于Apache Hadoop, Flink也有`InputFormat`和`OutputFormat`的概念.

`HadoopInputFormat`就是`InputFormat`的一种实现.
这是一种包装器,允许用户在Flink中使用所有存在的Hadoop input转换器.

这个章节展示了一些Flink连接到其他系统的例子.
[阅读更多有关Flink和Hadoop兼容性的内容]({{ site.baseurl }}/dev/batch/hadoop_compatibility.html).

## Flink对于Avro的支持

Flink 有大量的插件支持[Apache Avro](http://avro.apache.org/). 这使得Flink对Avro文件的读取变得非常容易.
并且, Flink的序列化框架能够操作Avro生成的class文件.只要在你工程中的pom.xml文件中添加了Flink Avro的依赖.

~~~xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
~~~

为了从Avro文件中读取数据, 你需要指定一个`AvroInputFormat`.

**例如**:

~~~java
AvroInputFormat<User> users = new AvroInputFormat<User>(in, User.class);
DataSet<User> usersDS = env.createInput(users);
~~~

注意,`User`是Avro生成的一个POJO. Flink也允许通过POJO的名字来获取. 例如:

~~~java
usersDS.groupBy("name")
~~~


注意在使用`GenericData`的时候.Record类型也是可以的,但是并不推荐.由于记录包含完整的schema, 因此它的数据是非常集中的,这可能导致用起来很慢.

Flink的POJO文件也可以和Avro生成的POJOs一起工作. 然而,只有当字段类型被正确的写入生成的class,才是可用的.如果一个字段是`Object`类型,你不能将这个字段作为一个join或者grouping的key.
在Avro中指定一个字段大概就是这样子的 `{"name": "type_double_test", "type": "double"},` 就可以正常使用了.然而指定如果指定UNION类型在仅有的一个字段中(`{"name": "type_double_test", "type": ["double"]},`)将导致变为`Object`类型. 注意指定null类型也是可以的! (`{"name": "type_double_test", "type": ["null", "double"]},`)!



### 微软Access数据库

_小提示: 下面的例子从Flink 0.6-incubating版本生效_

这个例子使用了`HadoopInputFormat`包装器,实现一个已有的Hadoop输入格式化接口,使[Azure表存储](https://azure.microsoft.com/en-us/documentation/articles/storage-introduction/)可用.

1. 下载并编译 `azure-tables-hadoop` 工程. 这个输入格式化还没有放入到Maven中央仓库中, 因此, 我们需要自己编译它.
执行下面的命令:

   ~~~bash
   git clone https://github.com/mooso/azure-tables-hadoop.git
   cd azure-tables-hadoop
   mvn clean install
   ~~~

2. 使用下面的向导,安装一个新的Flink工程 :

   ~~~bash
   curl https://flink.apache.org/q/quickstart.sh | bash
   ~~~

3. 将下面的依赖添加到你的`pom.xml`文件中 (在`<dependencies>`的作用域中):

   ~~~xml
   <dependency>
       <groupId>org.apache.flink</groupId>
       <artifactId>flink-hadoop-compatibility{{ site.scala_version_suffix }}</artifactId>
       <version>{{site.version}}</version>
   </dependency>
   <dependency>
     <groupId>com.microsoft.hadoop</groupId>
     <artifactId>microsoft-hadoop-azure</artifactId>
     <version>0.0.4</version>
   </dependency>
   ~~~

   `flink-hadoop-compatibility` 是一个Flink包,它提供了Hadoop输入格式化包装器.
   `microsoft-hadoop-azure` 将之前我们编译的工程添加到这个工程中.

现在,启动工程的代码已经准备就绪. 我们建议将工程引入一个IDE,比如Eclipse或者IntelliJ. (作为一个Maven工程引入!).
打开`Job.java`这个文件. 这个一个空的Flink job框架.

将下面的代码粘贴到这个文件中:

~~~java
import java.util.Map;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.hadoopcompatibility.mapreduce.HadoopInputFormat;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import com.microsoft.hadoop.azure.AzureTableConfiguration;
import com.microsoft.hadoop.azure.AzureTableInputFormat;
import com.microsoft.hadoop.azure.WritableEntity;
import com.microsoft.windowsazure.storage.table.EntityProperty;

public class AzureTableExample {

  public static void main(String[] args) throws Exception {
    // set up the execution environment
    final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

    // create a  AzureTableInputFormat, using a Hadoop input format wrapper
    HadoopInputFormat<Text, WritableEntity> hdIf = new HadoopInputFormat<Text, WritableEntity>(new AzureTableInputFormat(), Text.class, WritableEntity.class, new Job());

    // set the Account URI, something like: https://apacheflink.table.core.windows.net
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.ACCOUNT_URI.getKey(), "TODO");
    // set the secret storage key here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.STORAGE_KEY.getKey(), "TODO");
    // set the table name here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.TABLE_NAME.getKey(), "TODO");

    DataSet<Tuple2<Text, WritableEntity>> input = env.createInput(hdIf);
    // a little example how to use the data in a mapper.
    DataSet<String> fin = input.map(new MapFunction<Tuple2<Text,WritableEntity>, String>() {
      @Override
      public String map(Tuple2<Text, WritableEntity> arg0) throws Exception {
        System.err.println("--------------------------------\nKey = "+arg0.f0);
        WritableEntity we = arg0.f1;

        for(Map.Entry<String, EntityProperty> prop : we.getProperties().entrySet()) {
          System.err.println("key="+prop.getKey() + " ; value (asString)="+prop.getValue().getValueAsString());
        }

        return arg0.f0.toString();
      }
    });

    // emit result (this works only locally)
    fin.print();

    // execute program
    env.execute("Azure Example");
  }
}
~~~

下面的例子展示了如何使用一张Azure表并且将数据导入到Flink的`DataSet`中.(强调一下,set的类型应该是`DataSet<Tuple2<Text, WritableEntity>>`). 通过这个`DataSet`, 你可以将所有已知的转换应用到DataSet中去.

## 使用MongoDB

[GitHub文档描述了如何在MongoDB中使用Apache Flink (从0.7-incubating)版本开始](https://github.com/okkam-it/flink-mongodb-test).
