## 集群模式

Flink程序可以在集群环境中以分布式的方式执行，目前有两种将Flink程序提交到集群中的方法：

### 命令行接口

命令行接口允许你将打好的JAR包提交到集群（包括由一台服务器组成的集群）中。

如果想进步一步了解命令行接口的有关细节，请参考命令行接口的有关文档。网址如下：

https://ci.apache.org/projects/flink/flink-docs-release-1.2/setup/cli.html

### 远程执行环境

远程执行环境允许你直接在集群中执行执行由Java编写的Flink程序。在这种接口方式下，我们需要在程序中配置一个指向远端Flink集群的地址。

### Maven依赖包

如果你使用Maven来管理你Flink程序的一来关系，请在pom.xml文件中加入如下所示的flink-clients依赖模块。

```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-clients_2.10</artifactId>
  <version>1.2.0</version>
</dependency>

```

### 代码示例

如下代码演示了远程执行环境的使用方法

```
public static void main(String[] args) throws Exception {
    ExecutionEnvironment env = ExecutionEnvironment
        .createRemoteEnvironment("flink-master", 6123, "/home/user/udfs.jar");

    DataSet<String> data = env.readTextFile("hdfs://path/to/file");

    data
        .filter(new FilterFunction<String>() {
            public boolean filter(String value) {
                return value.startsWith("http://");
            }
        })
        .writeAsText("hdfs://path/to/result");

    env.execute();
}
```
注：上述代码中包含了用户应用相关的代码，因此需要包含特定Class文件的JAR包才能执行，这个JAV包的路径在调用类ExecutionEnvironment的createRemoteEnvironment函数以创建该类的实例时指定。
