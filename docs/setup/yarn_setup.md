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

## ���ٿ�ʼ

### ��yarn������һ�����ڵ�flink��Ⱥ

����һ��ӵ��4��task��yarn�Ự��ÿ��task��4gb�Ķ��ڴ�:

```
# ��flink����ҳ��ȡhaddoop2��
# http://flink.apache.org/downloads.html
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/yarn-session.sh -n 4 -jm 1024 -tm 4096
```

�ر�ָ����-s������ʾÿ��task�Ͽ��õĴ���ڵ����������ǽ���ѽڵ��������ó�ÿ�������Ĵ�����������
һ���Ự�������������ʹ��./bin/flink�����ύ���񵽼�Ⱥ�ϡ�

### ��yarn������һ��flink������

```
# ��flink����ҳ��ȡhaddoop2��
# http://flink.apache.org/downloads.html
curl -O <flink_hadoop2_download_url>
tar xvzf flink-{{ site.version }}-bin-hadoop2.tgz
cd flink-{{ site.version }}/
./bin/flink run -m yarn-cluster -yn 4 -yjm 1024 -ytm 4096 ./examples/batch/WordCount.jar
```

## flink yarn �Ự

apache [Hadoop YARN](http://hadoop.apache.org/)��һ����Դ����ܹ�������һ����Ⱥ�����ж���ֲ�ʽӦ�á�
flink��yarn���൱������Ӧ�á�����Ѿ���yarn�������ˣ��û��������û�װ�κζ�����

**ֻ��**

- hadoop�汾����2.2
- HDFS��hadoop�ֲ�ʽ�ļ�ϵͳ������������hadoop֧�ֵķֲ�ʽ�ļ�ϵͳ��.

�������ʹ��flink yarn�ͻ���������ʱ���뿴��[������̳](http://flink.apache.org/faq.html#yarn-deployment).

### ����flink�Ự

�������½���ѧϰ���������yran��Ⱥ������һ��flink�Ự.

һ���Ự����������flink����job��������task����������������Ϳ����ύ�������Ⱥ���У���ס��һ���Ự�п�������
�������

#### ����flink

����һ��hadoop�汾����2��flink�����ɴӸ�����ҳ��á���������������ļ���
��ȡ���ذ��ķ���:

```
tar xvzf flink-1.4-SNAPSHOT-bin-hadoop2.tgz
cd flink-1.4-SNAPSHOT/
```

#### ����һ���Ự

ʹ����������������һ���Ự��

```
./bin/yarn-session.sh
```

������ĸ������£�

```
ʹ��:
   ��������
     -n,--container <arg>   yarn���������� (=taskmanager�ĸ�����
   ��ѡ����
     -D <arg>                        ��̬����
     -d,--detached                   �������루�ύjob�Ļ�����yarn��Ⱥ���룩
     -jm,--jobManagerMemory <arg>    JobManager Container�ڴ��С [in MB]
     -nm,--name                      �Զ����ύjob������
     -q,--query                      չʾyarn�Ŀ�����Դ���ڴ�ͺ��� (memory, cores)
     -qu,--queue <arg>               ָ��yarn����.
     -s,--slots <arg>                ÿ��TaskManager�Ľڵ���
     -tm,--taskManagerMemory <arg>   ÿ��TaskManager Container���ڴ��С [in MB]
     -z,--zookeeperNamespace <arg>   �ڸ߿���ģʽ�£������ռ�Ϊzookeeper������·��
```

��ע��ͻ��˱����YARN_CONF_DIR or HADOOP_CONF_DIR�����������������ã����Զ�ȡ��yarn��hdfs���á�

**����:** �����������10��task manager��ÿ��taskӵ��8GB�ڴ��32������ڵ㣺

```
./bin/yarn-session.sh -n 10 -tm 8192 -s 32
```

ϵͳ��ʹ��conf/flink-conf.yaml�µ����á�����������һЩ���ã���ο������ֲᡣ

flink��yarn�ϣ�������д�������ò�����ֵ��jobmanager.rpc.address����Ϊjob manager���Ƿ����ڲ�ͬ�����ϣ���
taskmanager.tmp.dirs������ʹ��yarn����tmpĿ¼����parallelism.default������ڵ������ָ������

����㲻��ı������ļ����������ò����������и���������ö�̬���ԣ�ͨ��-D��ʾ����������ͨ�����·��������ݲ�����
-Dfs.overwrite-files=true -Dtaskmanager.network.memory.min=536346624.

���ӽ���������11�����������ܽ���10������������Ϊ����Ҫ�����1��������ApplicationMaster and Job Manager.

ֻҪflink������yarn��Ⱥ�ϣ��������㿴��Job Manager�������ϸ�ڡ�

ͨ��ֹͣunix���̣�ʹ��CTRL+C�����ֹͣyarn�Ự�������ڿͻ�������stop��

flink��yran�Ͻ�����������������������yarn��Ⱥ�����㹻�Ŀ�����Դ�����yarn���ȳ���Ϊ���������������ڴ棬һЩ������vcores������

Ĭ�������vcores�������ڴ���ڵ�����-s����yarn.containers.vcores�����Զ���ֵ��дvcores������

#### ����yarn�Ự

If you do not want to keep the Flink YARN client running all the time, it's also possible to start a *detached* YARN session.
����㲻�뱣��flink yarn�ͻ���һֱ���У�������������yarn�Ự���ﵽĿ�ġ������������-d��--detached��
�ڴ�����£�flink yarn�ͻ��˽����ύflink����Ⱥ�У�Ȼ��ر����ӡ�ע������ڴ�����£���������ʹ��flink��ֹͣyarn�Ự��
ʹ��yarn���yarn application --kill <appId>����ֹͣyarn�Ự��

#### �������лỰ

ʹ��������������һ���Ự

```
./bin/yarn-session.sh
```
������չʾ���¸�����

```
ʹ��:
  �ر�
     -id,--applicationId <yarnAppId> YARN application Id
```
��֮ǰ������YARN_CONF_DIR �� HADOOP_CONF_DIR������������������YARN �� HDFS ���ö�ȡ����

**����:** ���������������һ�������е�flink yarn�Ựapplication_1463870264508_0029

```
./bin/yarn-session.sh -id application_1463870264508_0029
```

ʹ��yarn ��Դ������������job��������RPC�˿ڴӶ�����һ�����еĻỰ��
ֹͣyarn�Ự��ͨ��ֹͣunix���̣�CTRL+C����ͨ���ٿͻ�������stop��

### �ύjob��flink

ʹ�����������ύһ��flink����yarn��Ⱥ��

```
./bin/flink
```

��ο������пͻ����ĵ���
�����а����˵����£�

```
[...]
run������������г���

 �﷨: run [OPTIONS] <jar-file> <arguments>
  "run" ��������:
     -c,--class <classname>           ������ڵ��� ("main"
                                      ���� �� "getPlan()" ����.jar�ļ�û�������嵥��ָ�������Ҫ.
     -m,--jobmanager <host:port>      ����job��������master���ĵ�ַ. ʹ�ô˲�������һ����ͬ��job����������������������ָ��.
     -p,--parallelism <parallelism>   ���г���Ĳ��ж�. �����ѡ�����ɸ���������ָ����Ĭ��ֵ��
```

��run�����ύһ��job��yarn�ϡ��ͻ��˿��Ծ���job�������ĵ�ַ����������£����ʹ��-m����ָ��job��������ַ��job��������ַ����yarn����̨������

**����**

```
wget -O LICENSE-2.0.txt http://www.apache.org/licenses/LICENSE-2.0.txt
hadoop fs -copyFromLocal LICENSE-2.0.txt hdfs:/// ...
./bin/flink run ./examples/batch/WordCount.jar \
        hdfs:///..../LICENSE-2.0.txt hdfs:///.../wordcount-result.txt
```
����������´�����ȷ������task�������Ѿ�����:

```
Exception in thread "main" org.apache.flink.compiler.CompilerException:
    Available instances could not be determined from job manager: Connection timed out.
```

�������job��������web�ӿ��в鿴task���������������ӿڵĵ�ַ����yarn�Ự�Ŀ���̨�������
���task������һ������û����ʾ������ô��Ӧ������־�ļ��м��������ġ�

## ��yarn������һ��flink ����

�����ĵ��������������һ��flink��Ⱥ��hadoop yarn�����¡���Ҳ���Խ�ִ��һ��job������flink��yarn�¡�
��ע��ͻ�����Ҫ-ynֵ������task��������������

***����:***

```
./bin/flink run -m yarn-cluster -yn 2 ./examples/batch/WordCount.jar
```

��yarn�Ự�������� ./bin/flink tool�ǿ�ѡ�ģ���y��yarnǰ׺��

ע�⣺�����ͨ������FLINK_CONF_DIR����������Ϊÿ��jobʹ�ò�ͬ������Ŀ¼��
ʹ���������������flink�ֲ���confĿ¼��������ÿ��job����־��

ע�⣺���-m yarn-cluster�͸���yarn�Ự��-yd�������"�ٻٺ�����"�ύflink job��yarn��Ⱥ�С�
�ڴ�����£����Ӧ�ó��򽫵ò����κ�ȷ�Ͻ���� �ų�ExecutionEnvironment.execute()��������Ϣ��

## ʹ��jars&Classpath

Ĭ���£�flink����õ���jars����ϵͳ·����������һ��jobʱ�������Ϊ������yarn.per-job-cluster.include-user-jar
���������ơ�

�������������ΪDISABLEDʱ��flink�����û�·����jars������

user-jars��ϵͳ·��λ�ÿ���ͨ�����ò��������ƣ�
- ORDER��Ĭ�ϣ������ֵ�·��˳�����jar��ϵͳ��
- FIRST:ϵͳ·����ǰ����ӡ�
- LAST:ϵͳ·��������ӡ�

## flink��yarn�ϵĻָ���Ϊ

flink��yarn�ͻ������������ò�����������Ϊ������ʧ�ܺ���Щ������ͨ��conf/flink-conf.yaml���ã�Ҳ����ͨ��
������yarn�Ựʱ��-D�������á�

- `yarn.reallocate-failed`: ����flink�Ƿ����·���ʧ�ܵ�task��������Ĭ��true��
- `yarn.maximum-failed-containers`: applicationMaster���ܵ��������ʧ�ܸ�����ֱ��yarn�Ựʧ�ܡ�Ĭ����-n���õ�task������������
- `yarn.application-attempts`: applicationMaster��+��ӵ�е�task�������������ĳ��Դ�����Ĭ��1��applicationMasterʧ����yarn�Ự����ʧ�ܡ���yarn��ָ������ֵ�Ա�����applicationMaster��

## ����һ��ʧ�ܵ�yarn�Ự

�кܶ�ԭ��ʹ��һ��flink��yarn�Ựʧ�ܡ�һ�������hadoop��װ��hdfsȨ�ޣ�yarn���ã����汾���ݣ�����flink��vanilla��hadoop�ϣ�ȴ����Cloudera Hadoop��������ԭ��

### ��־�ļ�

����ʱflink yarn�Ựʧ�ܣ��û���������hadoop yarn����־��
�����õ���yarn��־���ϡ��û�������yarn-site.xml�ļ��а�yarn.log-aggregation-enable����ֵ����Ϊtrue��
ʹ����Ч��ֻҪ��һ����Ч���û�����ʹ����������������һ����ʧ�ܣ�yarn�Ự��������־�ļ���

```
yarn logs -applicationId <application ID>
```

�ڻỰ����ʱ��ȴ�������ֱ����־չʾ������

### yarn�ͻ��˿���̨&web�ӿ�

flink yarn�ͻ���Ҳ�������ն����������Ϣ�����������ʱ������ĳʱ��task������ֹͣ������.���⣬��yarn ��Դ��������web�ӿڣ�Ĭ����8088�˿ڣ��������Դ������web�ӿڵĶ˿���
yarn.resourcemanager.webapp.address����ֵ������

��webҳ��ɷ�������yarnӦ�ó������־�ļ�������ʾʧ��Ӧ�ó���������Ϣ��

Ϊָ��hadoop�汾����yarn�ͻ���

�û�ʹ����Hortonworks, Cloudera or MapR�ȹ�˾������hadoop�����ǵ�hadoop��hdfs���汾��yarn�汾�����빹��flink��ͻ��
��ο��������ܻ�ø�ϸ���ܡ�

## Build YARN client for a specific Hadoop version

Users using Hadoop distributions from companies like Hortonworks, Cloudera or MapR might have to build Flink against their specific versions of Hadoop (HDFS) and YARN. Please read the [build instructions](building.html) for more details.

## ����ǽ����yarn����flink

һЩyarn��Ⱥʹ�÷���ǽ�����Ƽ�Ⱥ����������֮������紫�䣬�����������£�flink��job�ύ��yarn�Ự��ֻ��ͨ����Ⱥ���磨�ڷ���ǽ���󣩣�
��������������²����У�flink��������һ����Χ�Ķ˿ڸ���ط���
����Щ��Χ�����£��û����Կ�Խ����ǽ�ύjob��flink��

��ǰ��������������Ҫ�ύjob:

 * job��������yarn�ϵ�ApplicationMaster��
 * ����job��������BlobServer

���ύһ��job��flink��BlobServer����ַ��û������е�jars�����й����ڵ㣨task����������
job����������job��������ִ�С�

�����������ò�����ָ���˿�:
 * `yarn.application-master.port`
 * `blob.server.port`

���������ÿɽ��յ����˿�ֵ����50010����Ҳ���Խ��շ�Χ��50000-50025��������
��ϣ�50010,50011,50020-50025,50050-50075��

��hadoopʹ��ͬ���Ļ��ƣ����ò�����yarn.app.mapreduce.am.job.client.port-range��

## ����/�ڲ�

��С�ڼ�Ҫ����flink��yarn��ν���.

<img src="{{ site.baseurl }}/fig/FlinkOnYarn.svg" class="img-responsive">

yarn�ͻ�����Ҫ����hadoop������������yarn��Դ��������hdfs���������hadoop���ò�ȡ���²��ԣ�

* ����YARN_CONF_DIR, HADOOP_CONF_DIR or HADOOP_CONF_PATH ������˳���Ƿ������ã�����һ�������ˣ����ǾͿ��Զ�ȡ�����á�
* ������������ʧ�ܣ���ȷ��yarn��װ������ִ���������ͻ���ʹ��HADOOP_HOME�����������绷�����������ˣ��ͻ��˻᳢�Է���$HADOOP_HOME/etc/hadoop��hadoop2.*���� $HADOOP_HOME/conf��hadoop1.*��

������һ���µ�flink yarn�Ự���ͻ��˻���ȷ���������Դ���������ڴ棩�Ƿ��ܻ�õ���
֮�󣬿ͻ����ϴ�����flink��hdfs���õ�jars������1����

��һ���ͻ�������һ��yarn����������2��������ApplicationMaster������3����
�ͻ���ע�������ú�������Դ��jar�ļ���ָ���������е�yarn�ڵ��������׼���������������ļ�����
��Щ�����ˣ�ApplicationMaster (AM)�������ˡ�

job��������AM������ͬһ����������ǳɹ�������AM֪��job����������ӵ�е��������ĵ�ַ��

job������Ϊtask����������һ���µ�flink���ã�����task������job����������

�ļ�Ҳ�ϴ���hdfs�ϡ�����AM����ҲΪflink��web�ӿڷ���yarn��������ж˿��Ƿ������ʱ�˿ڡ�
������û�����ִ�ж��yarn�Ự��

Ȼ��AM�������䵽����������Щ������flink��task����������������jar�͸�������hdfs����
����Щ������ɺ�flink�Ͱ�װ�����ˣ����Խ���job�ˡ�