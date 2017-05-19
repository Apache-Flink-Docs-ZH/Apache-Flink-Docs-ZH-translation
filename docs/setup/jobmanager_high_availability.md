---
title: "JobManager的高可用 (HA)"
nav-title: High Availability (HA)
nav-parent_id: setup
nav-pos: 6
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
JobManager协调每个Flink部署。 它负责 *调度* 和 *资源管理*。 

默认情况下，每个Flink集群都只一个JobManager节点。 
这样的架构会有*单点故障*（SPOF）：如果JobManager挂了，则不能提交新的程序，且运行中的程序会失败。 

使用JobManager高可用性，集群可以从JobManager故障中恢复，从而避免* SPOF *。 
用户在**standalone**或** YARN集群**模式下，配置高可用性。 

* Toc
{:toc}

## Standalone模式下集群的高可用

Standalone模式（独立模式）下JobManager的高可用性的基本思想是，任何时候都有一个**一个 Master JobManager **，并且**多个Standby JobManagers **。
Standby JobManagers可以在Master JobManager 挂掉的情况下接管集群成为Master JobManager。 
这样保证了**没有单点故障**，一旦某一个Standby JobManager接管集群，程序就可以继续运行。 
Standby JobManager和Master JobManager实例之间没有明确区别。 每个JobManager可以成为Master或Standby节点。

举例，使用三个JobManager节点的情况下，进行以下设置:

<img src="{{ site.baseurl }}/fig/jobmanager_ha_overview.png" class="center" />

### 配置

要启用JobManager高可用性，必须将**高可用性模式**设置为* zookeeper *，
配置一个** ZooKeeper quorum**，并配置一个**主文件** 存储所有JobManager hostname 及其Web UI端口号。 

Flink利用** [ZooKeeper]（http://zookeeper.apache.org）** 实现运行中的JobManager节点之间的*分布式协调*。 ZooKeeper是独立于Flink的服务，它通过领导选举制和轻量级状态一致性存储 来提供高度可靠的分布式协调。 
有关ZooKeeper的更多信息，请查看“ZooKeeper入门指南”（http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html）。 
Flink包含[安装简单ZooKeeper]（＃bootstrap-zookeeper）的bootstrap脚本。
 
#### Masters File (masters)

为了启动集群的高可用（HA），需要配置`conf/masters`中的* masters *文件  :

- **masters file**: *masters file* 文件中包含所有启动JobManager节点的hosts，以及对应Web UI的端口号 

  <pre>
jobManagerAddress1:webUIPort1
[...]
jobManagerAddressX:webUIPortX
  </pre>

默认情况下，JobManager会选择 *随机端口号* 作为内部进程通信。我们可以通过修改参数**recovery.jobmanager.port**的值来修改。这个参数的配置接受为单个端口号（比如50010），也可以设置一段范围比如 50000~50025，或者端口号组合（比如50010，50011，50020~50025,50050~50075）。

#### 配置文件(flink-conf.yaml)

为了启动集群的高可用（HA），需要对文件`conf/flink-conf.yaml`做一下配置： 

- **高可用模式** (必须的): 要启用高可用，需要配置文件 `conf/flink-conf.yaml` 中的 *high-availability mode* 必须设置为 *zookeeper*。

  <pre>high-availability: zookeeper</pre>

- **ZooKeeper quorum** (required): A *ZooKeeper quorum* is a replicated group of ZooKeeper servers, which provide the distributed coordination service.

  <pre>high-availability.zookeeper.quorum: address1:2181[,...],addressX:2181</pre>

  Each *addressX:port* refers to a ZooKeeper server, which is reachable by Flink at the given address and port.

- **ZooKeeper root** (recommended): The *root ZooKeeper node*, under which all cluster namespace nodes are placed.

  <pre>high-availability.zookeeper.path.root: /flink

- **ZooKeeper namespace** (recommended): The *namespace ZooKeeper node*, under which all required coordination data for a cluster is placed.

  <pre>high-availability.zookeeper.path.namespace: /default_ns # important: customize per cluster</pre>

  **Important**: if you are running multiple Flink HA clusters, you have to manually configure separate namespaces for each cluster. By default, the Yarn cluster and the Yarn session automatically generate namespaces based on Yarn application id. A manual configuration overrides this behaviour in Yarn. Specifying a namespace with the -z CLI option, in turn, overrides manual configuration.

- **Storage directory** (required): JobManager metadata is persisted in the file system *storageDir* and only a pointer to this state is stored in ZooKeeper.

    <pre>
high-availability.zookeeper.storageDir: hdfs:///flink/recovery
    </pre>

    The `storageDir` stores all metadata needed to recover a JobManager failure.

After configuring the masters and the ZooKeeper quorum, you can use the provided cluster startup scripts as usual. They will start an HA-cluster. Keep in mind that the **ZooKeeper quorum has to be running** when you call the scripts and make sure to **configure a separate ZooKeeper root path** for each HA cluster you are starting.

#### Example: Standalone Cluster with 2 JobManagers

1. **Configure high availability mode and ZooKeeper quorum** in `conf/flink-conf.yaml`:

   <pre>
high-availability: zookeeper
high-availability.zookeeper.quorum: localhost:2181
high-availability.zookeeper.path.root: /flink
high-availability.zookeeper.path.namespace: /cluster_one # important: customize per cluster
high-availability.zookeeper.storageDir: hdfs:///flink/recovery</pre>

2. **Configure masters** in `conf/masters`:

   <pre>
localhost:8081
localhost:8082</pre>

3. **Configure ZooKeeper server** in `conf/zoo.cfg` (currently it's only possible to run a single ZooKeeper server per machine):

   <pre>server.0=localhost:2888:3888</pre>

4. **Start ZooKeeper quorum**:

   <pre>
$ bin/start-zookeeper-quorum.sh
Starting zookeeper daemon on host localhost.</pre>

5. **Start an HA-cluster**:

   <pre>
$ bin/start-cluster.sh
Starting HA cluster with 2 masters and 1 peers in ZooKeeper quorum.
Starting jobmanager daemon on host localhost.
Starting jobmanager daemon on host localhost.
Starting taskmanager daemon on host localhost.</pre>

6. **Stop ZooKeeper quorum and cluster**:

   <pre>
$ bin/stop-cluster.sh
Stopping taskmanager daemon (pid: 7647) on localhost.
Stopping jobmanager daemon (pid: 7495) on host localhost.
Stopping jobmanager daemon (pid: 7349) on host localhost.
$ bin/stop-zookeeper-quorum.sh
Stopping zookeeper daemon (pid: 7101) on host localhost.</pre>

## YARN Cluster High Availability

When running a highly available YARN cluster, **we don't run multiple JobManager (ApplicationMaster) instances**, but only one, which is restarted by YARN on failures. The exact behaviour depends on on the specific YARN version you are using.

### Configuration

#### Maximum Application Master Attempts (yarn-site.xml)

You have to configure the maximum number of attempts for the application masters for **your** YARN setup in `yarn-site.xml`:

{% highlight xml %}
<property>
  <name>yarn.resourcemanager.am.max-attempts</name>
  <value>4</value>
  <description>
    The maximum number of application master execution attempts.
  </description>
</property>
{% endhighlight %}

The default for current YARN versions is 2 (meaning a single JobManager failure is tolerated).

#### Application Attempts (flink-conf.yaml)

In addition to the HA configuration ([see above](#configuration)), you have to configure the maximum attempts in `conf/flink-conf.yaml`:

<pre>yarn.application-attempts: 10</pre>

This means that the application can be restarted 10 times before YARN fails the application. It's important to note that `yarn.resourcemanager.am.max-attempts` is an upper bound for the application restarts. Therfore, the number of application attempts set within Flink cannot exceed the YARN cluster setting with which YARN was started.

#### Container Shutdown Behaviour

- **YARN 2.3.0 < version < 2.4.0**. All containers are restarted if the application master fails.
- **YARN 2.4.0 < version < 2.6.0**. TaskManager containers are kept alive across application master failures. This has the advantage that the startup time is faster and that the user does not have to wait for obtaining the container resources again.
- **YARN 2.6.0 <= version**: Sets the attempt failure validity interval to the Flinks' Akka timeout value. The attempt failure validity interval says that an application is only killed after the system has seen the maximum number of application attempts during one interval. This avoids that a long lasting job will deplete it's application attempts.

<p style="border-radius: 5px; padding: 5px" class="bg-danger"><b>Note</b>: Hadoop YARN 2.4.0 has a major bug (fixed in 2.5.0) preventing container restarts from a restarted Application Master/Job Manager container. See <a href="https://issues.apache.org/jira/browse/FLINK-4142">FLINK-4142</a> for details. We recommend using at least Hadoop 2.5.0 for high availability setups on YARN.</p>

#### Example: Highly Available YARN Session

1. **Configure HA mode and ZooKeeper quorum** in `conf/flink-conf.yaml`:

   <pre>
high-availability: zookeeper
high-availability.zookeeper.quorum: localhost:2181
high-availability.zookeeper.storageDir: hdfs:///flink/recovery
high-availability.zookeeper.path.root: /flink
high-availability.zookeeper.path.namespace: /cluster_one # important: customize per cluster
yarn.application-attempts: 10</pre>

3. **Configure ZooKeeper server** in `conf/zoo.cfg` (currently it's only possible to run a single ZooKeeper server per machine):

   <pre>server.0=localhost:2888:3888</pre>

4. **Start ZooKeeper quorum**:

   <pre>
$ bin/start-zookeeper-quorum.sh
Starting zookeeper daemon on host localhost.</pre>

5. **Start an HA-cluster**:

   <pre>
$ bin/yarn-session.sh -n 2</pre>

## Configuring for Zookeeper Security

If ZooKeeper is running in secure mode with Kerberos, you can override the following configurations in `flink-conf.yaml` as necessary:

<pre>
zookeeper.sasl.service-name: zookeeper     # default is "zookeeper". If the ZooKeeper quorum is configured
                                           # with a different service name then it can be supplied here.
zookeeper.sasl.login-context-name: Client  # default is "Client". The value needs to match one of the values
                                           # configured in "security.kerberos.login.contexts".
</pre>

For more information on Flink configuration for Kerberos security, please see [here]({{ site.baseurl}}/setup/config.html).
You can also find [here]({{ site.baseurl}}/ops/security-kerberos.html) further details on how Flink internally setups Kerberos-based security.

## Bootstrap ZooKeeper

If you don't have a running ZooKeeper installation, you can use the helper scripts, which ship with Flink.

There is a ZooKeeper configuration template in `conf/zoo.cfg`. You can configure the hosts to run ZooKeeper on with the `server.X` entries, where X is a unique ID of each server:

<pre>
server.X=addressX:peerPort:leaderPort
[...]
server.Y=addressY:peerPort:leaderPort
</pre>

The script `bin/start-zookeeper-quorum.sh` will start a ZooKeeper server on each of the configured hosts. The started processes start ZooKeeper servers via a Flink wrapper, which reads the configuration from `conf/zoo.cfg` and makes sure to set some required configuration values for convenience. In production setups, it is recommended to manage your own ZooKeeper installation.
