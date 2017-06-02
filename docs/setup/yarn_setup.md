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

### 在yarn上启动一个长期的flink集群

启动一个拥有4个task的yarn会话，每个task有4gb的堆内存:

```
# 从flink下载页获取haddoop2包
# http://flink.apache.org/downloads.html
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/yarn-session.sh -n 4 -jm 1024 -tm 4096
```

特别指出，-s参数表示每个task上可用的处理节点数量。我们建议把节点数量设置成每个机器的处理器个数。
一旦会话被启动，你可以使用./bin/flink工具提交任务到集群上。

### 在yarn上运行一个flink的任务

```
# 从flink下载页获取haddoop2包
# http://flink.apache.org/downloads.html
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/flink run -m yarn-cluster -yn 4 -yjm 1024 -ytm 4096 ./examples/batch/WordCount.jar
```

## flink yarn 会话

apache [Hadoop YARN](http://hadoop.apache.org/)是一个资源管理架构，允许一个集群上运行多个分布式应用。
flink在yarn上相当于其他应用。如果已经有yarn的配置了，用户不需配置或安装任何东西。

**只需**

- hadoop版本至少2.2
- HDFS（hadoop分布式文件系统）（或其他由hadoop支持的分布式文件系统）.

如果你在使用flink yarn客户端有问题时，请看此[问题论坛](http://flink.apache.org/faq.html#yarn-deployment).

### 启动flink会话

跟随以下介绍学习怎样在你的yran集群中启动一个flink会话.

一个会话将启动所有flink服务（job管理器和task管理器），这样你就可以提交程序给集群运行，记住在一个会话中可以运行
多个程序。

#### 下载flink

下载一个hadoop版本大于2的flink包，可从该下载页获得。它包含了所需的文件。
提取下载包的方法:

```
tar xvzf flink-1.4-SNAPSHOT-bin-hadoop2.tgz
cd flink-1.4-SNAPSHOT/
```

#### 启动一个会话

使用如下命令来启动一个会话，

```
./bin/yarn-session.sh
```

该命令的概览如下：

```
使用:
   参数必须
     -n,--container <arg>   yarn上容器个数 (=taskmanager的个数）
   可选参数
     -D <arg>                        动态属性
     -d,--detached                   启动分离（提交job的机器与yarn集群分离）
     -jm,--jobManagerMemory <arg>    JobManager Container内存大小 [in MB]
     -nm,--name                      自定义提交job的名字
     -q,--query                      展示yarn的可用资源，内存和核数 (memory, cores)
     -qu,--queue <arg>               指定yarn队列.
     -s,--slots <arg>                每个TaskManager的节点数
     -tm,--taskManagerMemory <arg>   每个TaskManager Container的内存大小 [in MB]
     -z,--zookeeperNamespace <arg>   在高可用模式下，命名空间为zookeeper创建子路径
```

请注意客户端必须的YARN_CONF_DIR or HADOOP_CONF_DIR环境变量参数的设置，可以读取到yarn和hdfs配置。

**例子:** 如下命令分配10个task manager，每个task拥有8GB内存和32个处理节点：

```
./bin/yarn-session.sh -n 10 -tm 8192 -s 32
```

系统将使用conf/flink-conf.yaml下的配置。如果你想更改一些配置，请参考配置手册。

flink在yarn上，将会重写如下配置参数的值，jobmanager.rpc.address（因为job manager总是分配在不同机器上），
taskmanager.tmp.dirs（我们使用yarn给的tmp目录），parallelism.default（如果节点个数被指定）。

如果你不想改变配置文件来设置配置参数，这里有个方法来获得动态属性，通过-D标示。这样可以通过以下方法来传递参数，
-Dfs.overwrite-files=true -Dtaskmanager.network.memory.min=536346624.

例子将请求启动11个容器（尽管仅需10个容器），因为这需要额外的1个容器给ApplicationMaster and Job Manager.

只要flink部署在yarn集群上，它会让你看到Job Manager间的连接细节。

通过停止unix进程（使用CTRL+C命令）来停止yarn会话，或者在客户端输入stop。

flink在yran上仅仅启动所请求的容器，如果yarn集群上有足够的可用资源。大多yarn调度程序为容器，计算请求内存，一些还计算vcores数量。

默认情况，vcores数量等于处理节点数（-s），yarn.containers.vcores允许自定义值重写vcores数量。

#### 隔离yarn会话

If you do not want to keep the Flink YARN client running all the time, it's also possible to start a *detached* YARN session.
如果你不想保持flink yarn客户端一直运行，可以启动隔离yarn会话来达到目的。这个参数即是-d或--detached。
在此情况下，flink yarn客户端将仅提交flink到集群中，然后关闭连接。注意的是在此情况下，将不可能使用flink来停止yarn会话。
使用yarn命令（yarn application --kill <appId>）来停止yarn会话。

#### 关联现有会话

使用如下命令启动一个会话

```
./bin/yarn-session.sh
```
这个命令将展示如下概览：

```
使用:
  必备
     -id,--applicationId <yarnAppId> YARN application Id
```
如之前所述，YARN_CONF_DIR 或 HADOOP_CONF_DIR环境变量需设置能让YARN 和 HDFS 配置读取到。

**例子:** 假设以下命令关联一个正运行的flink yarn会话application_1463870264508_0029

```
./bin/yarn-session.sh -id application_1463870264508_0029
```

使用yarn 资源管理器来决定job管理器的RPC端口从而关联一个运行的会话。
停止yarn会话可通过停止unix进程（CTRL+C）或通过再客户端输入stop。

### 提交job到flink

使用如下命令提交一个flink程序到yarn集群：

```
./bin/flink
```

请参考命令行客户端文档。
命令行帮助菜单如下：

```
[...]
run操作编译和运行程序。

 语法: run [OPTIONS] <jar-file> <arguments>
  "run" 操作参数:
     -c,--class <classname>           程序入口的类 ("main"
                                      方法 或 "getPlan()" 方法.jar文件没有在其清单中指定类才需要.
     -m,--jobmanager <host:port>      连接job管理器（master）的地址. 使用此参数连接一个不同的job管理器，而不是在配置中指明.
     -p,--parallelism <parallelism>   运行程序的并行度. 这个可选参数可覆盖配置中指定的默认值。
```

用run操作提交一个job到yarn上。客户端可以决定job管理器的地址。罕见情况下，你可使用-m参数指定job管理器地址。job管理器地址可在yarn控制台见到。

**例子**

```
wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt
hadoop fs -copyFromLocal LICENSE-2.0.txt hdfs:/// ...
./bin/flink run ./examples/batch/WordCount.jar \
        hdfs:///..../LICENSE-2.0.txt hdfs:///.../wordcount-result.txt
```
如果存在如下错误，请确保所有task管理器已经启动:

```
Exception in thread "main" org.apache.flink.compiler.CompilerException:
    Available instances could not be determined from job manager: Connection timed out.
```

你可以在job管理器的web接口中查看task管理器的数量。接口的地址会在yarn会话的控制台中输出。
如果task管理器一分钟内没有显示出，那么你应该在日志文件中检查错误在哪。

## 在yarn上运行一个flink 任务

上述文档描述了如何启动一个flink集群在hadoop yarn环境下。这也可以仅执行一个job而启动flink在yarn下。
请注意客户端需要-yn值来设置task管理器的数量。

***例子:***

```
./bin/flink run -m yarn-cluster -yn 2 ./examples/batch/WordCount.jar
```

在yarn会话下命令行 ./bin/flink tool是可选的，以y或yarn前缀。

注意：你可以通过设置FLINK_CONF_DIR环境变量来为每个job使用不同的配置目录。
使用这个将拷贝来自flink分布下conf目录，并更新每个job的日志。

注意：组合-m yarn-cluster和隔离yarn会话（-yd）命令可"焚毁和忘掉"提交flink job在yarn集群中。
在此情况下，你的应用程序将得不到任何确认结果或 排除ExecutionEnvironment.execute()的请求消息。

## 使用jars&Classpath

默认下，flink会把用到的jars带进系统路径，当运行一个job时。这个行为可以用yarn.per-job-cluster.include-user-jar
参数来控制。

当设置这个参数为DISABLED时，flink将把用户路径的jars带进。

user-jars在系统路径位置可以通过设置参数来控制：
- ORDER：默认，按照字典路径顺序添加jar进系统。
- FIRST:系统路径最前的添加。
- LAST:系统路径最后的添加。

## flink在yarn上的恢复行为

flink的yarn客户端有如下配置参数来控制行为当容器失败后，这些参数可通过conf/flink-conf.yaml设置，也可以通过
在启动yarn会话时用-D参数设置。

- `yarn.reallocate-failed`: 控制flink是否重新分配失败的task管理器。默认true。
- `yarn.maximum-failed-containers`: applicationMaster接受的最大容器失败个数，直到yarn会话失败。默认是-n设置的task管理器个数。
- `yarn.application-attempts`: applicationMaster（+其拥有的task管理器个数）的尝试次数，默认1，applicationMaster失败则yarn会话整个失败。在yarn中指定更大值以便重启applicationMaster。

## 调试一个失败的yarn会话

有很多原因使得一个flink的yarn会话失败。一个错误的hadoop安装（hdfs权限，yarn配置），版本兼容（运行flink在vanilla的hadoop上，却依赖Cloudera Hadoop）或其他原因。

### 日志文件

部署时flink yarn会话失败，用户必须依靠hadoop yarn的日志。
最有用的是yarn日志集合。用户必须在yarn-site.xml文件中把yarn.log-aggregation-enable参数值设置为true，
使其生效。只要它一经生效，用户可以使用如下命令来检索一个（失败）yarn会话的所有日志文件。

```
yarn logs -applicationId <application ID>
```

在会话结束时请等待几秒钟直到日志展示出来。

### yarn客户端控制台&web接口

flink yarn客户端也可以在终端输出错误信息，如果在运行时出错（如某时间task管理器停止工作）.此外，有yarn 资源管理器的web接口（默认是8088端口），这个资源管理器web接口的端口由
yarn.resourcemanager.webapp.address参数值决定。

在web页面可访问运行yarn应用程序的日志文件并可显示失败应用程序的诊断信息。

为指定hadoop版本构建yarn客户端

用户使用像Hortonworks, Cloudera or MapR等公司发布的hadoop，它们的hadoop（hdfs）版本和yarn版本可能与构建flink冲突，
请参考构建介绍获得更细介绍。

## Build YARN client for a specific Hadoop version

Users using Hadoop distributions from companies like Hortonworks, Cloudera or MapR might have to build Flink against their specific versions of Hadoop (HDFS) and YARN. Please read the [build instructions](building.html) for more details.

## 防火墙后在yarn运行flink

一些yarn集群使用防火墙来控制集群和余下网络之间的网络传输，在这种配置下，flink的job提交到yarn会话中只能通过集群网络（在防火墙背后），
如果在生产环境下不可行，flink允许配置一定范围的端口给相关服务，
在这些范围配置下，用户可以跨越防火墙提交job到flink。

当前，有两个服务需要提交job:

 * job管理器（yarn上的ApplicationMaster）
 * 运行job管理器的BlobServer

当提交一个job到flink，BlobServer将会分发用户代码中的jars给所有工作节点（task管理器），
job管理器接收job本身并触发执行。

以下两个配置参数可指定端口:
 * `yarn.application-master.port`
 * `blob.server.port`

这两个配置可接收单个端口值（如50010），也可以接收范围（50000-50025），或者
组合（50010,50011,50020-50025,50050-50075）

（hadoop使用同样的机制，配置参数是yarn.app.mapreduce.am.job.client.port-range）

## 背后/内部

本小节简要描述flink和yarn如何交互.

<img src="{{ site.baseurl }}/fig/FlinkOnYarn.svg" class="img-responsive">

yarn客户端需要访问hadoop的配置以连接yarn资源管理器和hdfs，这决定了hadoop配置采取如下策略，

* 测试YARN_CONF_DIR, HADOOP_CONF_DIR or HADOOP_CONF_PATH （按此顺序）是否已配置，其中一个配置了，它们就可以读取到配置。
* 如若上述策略失败（正确的yarn安装不会出现此情况），客户端使用HADOOP_HOME环境变量。如环境变量设置了，客户端会尝试访问$HADOOP_HOME/etc/hadoop（hadoop2.*）或 $HADOOP_HOME/conf（hadoop1.*）

当启动一个新的flink yarn会话，客户端会先确认请求的资源（容器和内存）是否能获得到。
之后，客户端上传包含flink和hdfs配置的jars（步骤1）。

下一步客户端请求一个yarn容器（步骤2）来启动ApplicationMaster（步骤3），
客户端注册了配置和容器资源的jar文件，指定机器运行的yarn节点管理器会准备好容器（下载文件），
这些结束了，ApplicationMaster (AM)就启动了。

job管理器和AM运行在同一个容器里，它们成功启动后，AM知道job管理器（它拥有的主机）的地址。

job管理器为task管理器生成一个新的flink配置（这样task可连接job管理器）。

文件也上传到hdfs上。另外AM容器也为flink的web接口服务。yarn代码的所有端口是分配的临时端口。
这可让用户并行执行多个yarn会话。

然后，AM启动分配到的容器，这些容器给flink的task管理器，将会下载jar和更新来自hdfs配置
，这些步骤完成后，flink就安装起来了，可以接收job了。