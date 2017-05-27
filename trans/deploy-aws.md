## Amazon Web Services (AWS)
AWS提供了能支持Flink运行的云计算服务。

### EMR: Elastic MapReduce

Amazon弹性MapReduce（Amazon EMR）是一种能实现快速Hadoop集群部署的Web服务。我们推荐使用这种方式在AWS上部署Flink，因为在这种方式下，AWS已经帮助大家配置好了跟Hadoop相关的设置。

#### 创建EMR集群
AWS提供的EMR文档中已经包含了一个创建并启动EMR集群的示例，链接如下：

http://docs.aws.amazon.com/ElasticMapReduce/latest/ManagementGuide/emr-gs-launch-sample-cluster.html

这个示例可以帮助大家创建任何EMR版本的集群。不过，为了使用Flink，你只需要在配置EMR时选择Core Hadoop就可以了，而不需要去配置诸如Hue等任何其他组件。
