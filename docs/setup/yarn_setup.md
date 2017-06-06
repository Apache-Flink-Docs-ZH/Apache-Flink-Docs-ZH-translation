---
title:  "YARN Setup"
nav-title: YARN
nav-parent_id: deployment
nav-pos: 2
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

## 快速开始

### 在YARN上启动一个长期的Flink集群

启动一个拥有4个Task Manager的yarn会话，每个Task Manager有4gb的堆内存:

~~~bash
# 从flink下载页获取haddoop2包
# http://flink.apache.org/downloads.html
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/yarn-session.sh -n 4 -jm 1024 -tm 4096
~~~

特别指出，-s参数表示每个Task Manager上可用的处理槽（processing slot）数量。我们建议把槽数量设置成每个机器处理器的个数。
一旦会话被启动，你可以使用./bin/flink工具提交任务到集群上。

### 在YARN上运行一个Flink的任务

~~~bash
# 从flink下载页获取haddoop2包
# http://flink.apache.org/downloads.html
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/flink run -m yarn-cluster -yn 4 -yjm 1024 -ytm 4096 ./examples/batch/WordCount.jar
~~~

## Flink YARN 会话

Apache [Hadoop YARN](http://hadoop.apache.org/)是一个资源管理框架，允许一个集群上运行多种分布式应用程序。
Flink 可以和其他应用程序一起在 YARN 上运行。如果已经启动了YARN，用户就不需再启动或安装任何东西。

**要求**

- Apache Hadoop版本至少2.2
- HDFS（Hadoop分布式文件系统）（或其他由Hadoop支持的分布式文件系统）.

如果你在使用Flink YARN客户端有问题时，请看此[问题论坛](http://flink.apache.org/faq.html#yarn-deployment).

### 启动Flink会话

跟随以下介绍学习怎样在你的yran集群中启动一个Flink会话.

一个会话将启动所有Flink服务（JobManager and TaskManagers），这样你就可以提交程序给集群运行，记住在一个会话中可以运行
多个程序。

#### 下载Flink

下载一个Hadoop版本大于2的Flink包，可从[该下载页](http://flink.apache.org/downloads.html)获得。它包含了所需的文件。
提取下载包的方法:

~~~bash
tar xvzf flink-1.4-SNAPSHOT-bin-hadoop2.tgz
cd flink-1.4-SNAPSHOT/
~~~

#### 启动一个会话

使用如下命令来启动一个会话，

~~~bash
./bin/yarn-session.sh
~~~

该命令的概览如下：

~~~bash
使用:
   要求:
     -n,--container <arg>   YARN上容器个数 (=taskmanager的个数）
   可选参数
     -D <arg>                        动态属性
     -d,--detached                   启动分离（提交job的机器与yarn集群分离）
     -jm,--jobManagerMemory <arg>    JobManager Container内存大小 [in MB]
     -nm,--name                      自定义提交job的名字
     -q,--query                      展示yarn的可用资源，内存和核数 (memory, cores)
     -qu,--queue <arg>               指定yarn队列.
     -s,--slots <arg>                每个TaskManager的处理槽数
     -tm,--taskManagerMemory <arg>   每个TaskManager Container的内存大小 [in MB]
     -z,--zookeeperNamespace <arg>   在高可用模式下，命名空间为zookeeper创建子路径
~~~

请注意，客户端需要 YARN_CONF_DIR 或 HADOOP_CONF_DIR 环境变量被设置好，可以通过它读取 YARN 和 HDFS 的配置。

**例子:** 如下命令分配10个Task Manager，每个拥有8GB内存和32个处理槽：

~~~bash
./bin/yarn-session.sh -n 10 -tm 8192 -s 32
~~~

系统将使用conf/flink-conf.yaml下的配置。如果你想更改一些配置，请参考配置手册。

Flink在YARN上，将会重写如下配置参数的值，jobmanager.rpc.address（因为Job Manager总是分配在不同机器上），
taskmanager.tmp.dirs（我们使用YARN给的tmp目录），parallelism.default（如果槽个数被指定）。

如果你不想改变配置文件来设置配置参数，这里有个方法来获得动态属性，通过-D标示。这样可以通过以下方法来传递参数，
-Dfs.overwrite-files=true -Dtaskmanager.network.memory.min=536346624.

例子将请求启动11个容器（尽管仅需10个容器），因为这需要额外的1个容器给ApplicationMaster and Job Manager.

只要Flink部署在YARN集群上，它会让你看到Job Manager间的连接细节。

通过停止unix进程（使用CTRL+C命令）来停止YARN会话，或者在客户端输入stop。

Flink在YARN上仅仅启动所请求的容器，如果YARN集群上有足够的可用资源。大多YARN调度程序为容器，计算请求内存，一些还计算vcores数量。

默认情况，vcores数量等于处理节点数（-s），yarn.containers.vcores允许自定义值重写vcores数量。

#### 隔离YARN会话

如果你不想保持Flink YARN客户端一直运行，可以启动隔离YARN会话来达到目的。这个参数即是-d或--detached。
在此情况下，Flink YARN客户端将仅提交Flink到集群中，然后关闭连接。注意的是在此情况下，将不可能使用Flink来停止YARN会话。
使用YARN命令（yarn application --kill <appId>）来停止YARN会话。

#### 关联现有会话

使用如下命令启动一个会话

~~~bash
./bin/yarn-session.sh
~~~
这个命令将展示如下概览：

~~~bash
参数必须:
     -id,--applicationId <yarnAppId> YARN application Id
~~~
如之前所述，YARN_CONF_DIR 或 HADOOP_CONF_DIR环境变量需设置能让YARN 和 HDFS 配置读取到。

**例子:** 假设以下命令关联一个正运行的Flink YARN会话application_1463870264508_0029

~~~bash
./bin/yarn-session.sh -id application_1463870264508_0029
~~~

使用YARN 资源管理器来决定Job Manager的RPC端口从而关联一个运行的会话。
停止YARN会话可通过停止unix进程（CTRL+C）或通过再客户端输入stop。

### 提交job到Flink

使用如下命令提交一个Flink程序到YARN集群：

~~~bash
./bin/flink
~~~

请参考命令行客户端文档。
命令行帮助菜单如下：

~~~bash
[...]
run操作编译和运行程序。

 语法: run [OPTIONS] <jar-file> <arguments>
  "run" 操作参数:
     -c,--class <classname>           程序入口的类 ("main"方法 或 "getPlan()" 方法.jar文件没有在其清单中指定类才需要.
     -m,--jobmanager <host:port>      连接Job Manager（master）的地址. 使用此参数连接一个不同的job管理器，而不是在配置中指明.
     -p,--parallelism <parallelism>   运行程序的并行度. 这个可选参数可覆盖配置中指定的默认值。
~~~

用run操作提交一个job到YARN上。客户端可以决定Job Manager的地址。罕见情况下，你可使用-m参数指定Job Manager地址。Job Manager地址可在YARN控制台见到。

**例子**

~~~bash
wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt
hadoop fs -copyFromLocal LICENSE-2.0.txt hdfs:/// ...
./bin/flink run ./examples/batch/WordCount.jar \
        hdfs:///..../LICENSE-2.0.txt hdfs:///.../wordcount-result.txt
~~~
如果存在如下错误，请确保所有Task Manager已经启动:

~~~bash
Exception in thread "main" org.apache.flink.compiler.CompilerException:
    Available instances could not be determined from job manager: Connection timed out.
~~~

你可以在Job Manager的web接口中查看Task Manager的数量。接口的地址会在YARN会话的控制台中输出。
如果Task Manager一分钟内没有显示出，那么你应该在日志文件中检查错误在哪。

## 在YARN上运行一个Flink 任务

上述文档描述了如何启动一个Flink集群在Hadoop YARN环境下。这也可以仅执行一个job而启动Flink在YARN下。
请注意客户端需要-yn值来设置Task Manager的数量。

***例子:***

~~~bash
./bin/flink run -m yarn-cluster -yn 2 ./examples/batch/WordCount.jar
~~~

在YARN会话下命令行 ./bin/flink tool是可选的，以y或yarn前缀。

注意：你可以通过设置FLINK_CONF_DIR环境变量来为每个job使用不同的配置目录。
使用这个将拷贝来自Flink分布下conf目录，并更新每个job的日志。

注意：组合-m yarn-cluster和隔离YARN会话（-yd）命令可"焚毁和忘掉"提交Flink job在YARN集群中。
在此情况下，你的应用程序将得不到任何确认结果或 排除ExecutionEnvironment.execute()的请求消息。

## 使用jars&Classpath

默认下，Flink会把用到的jars带进系统路径，当运行一个job时。这个行为可以用yarn.per-job-cluster.include-user-jar
参数来控制。

当设置这个参数为DISABLED时，Flink将把用户路径的jars带进。

user-jars在系统路径位置可以通过设置参数来控制：
- ORDER：默认，按照字典路径顺序添加jar进系统。
- FIRST:系统路径最前的添加。
- LAST:系统路径最后的添加。

## Flink在YARN上的恢复行为

Flink的YARN客户端有如下配置参数来控制行为当容器失败后，这些参数可通过conf/flink-conf.yaml设置，也可以通过
在启动YARN会话时用-D参数设置。

- `yarn.reallocate-failed`: 控制Flink是否重新分配失败的Task Manager。默认true。
- `yarn.maximum-failed-containers`: ApplicationMaster接受的最大容器失败个数，直到YARN会话失败。默认是-n设置的Task Manager个数。
- `yarn.application-attempts`: ApplicationMaster（+其拥有的Task Manager个数）的尝试次数，默认1，ApplicationMaster失败则YARN会话整个失败。在YARN中指定更大值以便重启ApplicationMaster。

## 调试一个失败的YARN会话

有很多原因使得一个Flink的YARN会话失败。一个错误的Hadoop安装（HDFS权限，YARN配置），版本兼容（运行Flink在vanilla的Hadoop上，却依赖Cloudera Hadoop）或其他原因。

### 日志文件

部署时Flink YARN会话失败，用户必须依靠Hadoop YARN的日志。
最有用的是YARN日志集合。用户必须在yarn-site.xml文件中把yarn.log-aggregation-enable参数值设置为true，
使其生效。只要它一经生效，用户可以使用如下命令来检索一个（失败）yarn会话的所有日志文件。

~~~bash
yarn logs -applicationId <application ID>
~~~

在会话结束时请等待几秒钟直到日志展示出来。

### YARN客户端控制台&web接口

Flink YARN客户端也可以在终端输出错误信息，如果在运行时出错（如某时间Task Manager停止工作）.此外，有YARN资源管理器的web接口（默认是8088端口），这个资源管理器web接口的端口由
yarn.resourcemanager.webapp.address参数值决定。

在web页面可访问运行YARN应用程序的日志文件并可显示失败应用程序的诊断信息。

## 为指定Hadoop版本构建YARN客户端

用户使用像Hortonworks, Cloudera or MapR等公司发布的Hadoop，它们的Hadoop（HDFS）版本和YARN版本可能与构建Flink冲突，
请参考[构建介绍](https://ci.apache.org/projects/flink/flink-docs-release-1.3/setup/building.html)获得更细介绍。

## 防火墙后在YARN运行Flink

一些YARN集群使用防火墙来控制集群和余下网络之间的网络传输，在这种配置下，Flink的job提交到YARN会话中只能通过集群网络（在防火墙背后），
如果在生产环境下不可行，Flink允许配置一定范围的端口给相关服务，
在这些范围配置下，用户可以跨越防火墙提交job到Flink。

当前，有两个服务需要提交job:

 * Job Manager（YARN上的ApplicationMaster）
 * 运行Job Manager的BlobServer

当提交一个job到Flink，BlobServer将会分发用户代码中的jars给所有工作节点（Task Manager），
Job Manager接收job本身并触发执行。

以下两个配置参数可指定端口:
 * `yarn.application-master.port`
 * `blob.server.port`

这两个配置可接收单个端口值（如50010），也可以接收范围（50000-50025），或者
组合（50010,50011,50020-50025,50050-50075）

（Hadoop使用同样的机制，配置参数是yarn.app.mapreduce.am.job.client.port-range）

## 背后/内部

本小节简要描述Flink和YARN如何交互.

<img src="{{ site.baseurl }}/fig/FlinkOnYarn.svg" class="img-responsive">

YARN客户端需要访问Hadoop的配置以连接YARN资源管理器和HDFS,这决定了Hadoop配置采取如下策略，

* 测试YARN_CONF_DIR, HADOOP_CONF_DIR or HADOOP_CONF_PATH （按此顺序）是否已配置，其中一个配置了，它们就可以读取到配置。
* 如若上述策略失败（正确的YARN安装不会出现此情况），客户端使用HADOOP_HOME环境变量。如环境变量设置了，客户端会尝试访问$HADOOP_HOME/etc/hadoop（hadoop2.*）或 $HADOOP_HOME/conf（hadoop1.*）

当启动一个新的Flink YARN会话，客户端会先确认请求的资源（容器和内存）是否能获得到。
之后，客户端上传包含Flink和HDFS配置的jars（步骤1）。

下一步客户端请求一个YARN容器（步骤2）来启动ApplicationMaster（步骤3），
客户端注册了配置和容器资源的jar文件，指定机器运行的YARN节点管理器会准备好容器（下载文件），
这些结束了，ApplicationMaster (AM)就启动了。

Job Manager和AM运行在同一个容器里，它们成功启动后，AM知道job管理器（它拥有的主机）的地址。

Job Manager为Task Manager生成一个新的Flink配置（这样task可连接Job Manager）。

文件也上传到HDFS上。另外AM容器也为Flink的web接口服务。YARN代码的所有端口是分配的临时端口。
这可让用户并行执行多个yarn会话。

然后，AM启动分配到的容器，这些容器给Flink的Task Manager，将会下载jar和更新来自HDFS配置
，这些步骤完成后，Flink就安装起来了，可以接收job了。