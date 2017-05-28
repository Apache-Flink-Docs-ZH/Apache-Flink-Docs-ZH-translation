---
title: "Savepoints"
nav-parent_id: setup
nav-pos: 8
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

* toc
{:toc}
# 保存点
## 概况
保存点（savepoint）是用于恢复和更新 Flink 作业而特定保存的检查点。保存点（savepoint）使用 Flink 的检查点机制来创建程序以及相应状态的一个快照，并把快照保存到外存中。
当前页面包含了触发、还原以及处理保存点（savepoint）的步骤。为了能够在作业的不同版本之间以及 Flink 的不同版本之间顺利升级，需要重点的阅读 [给算子赋 ID](#assigning-operator-ids) 这一小节。
想了解更多关于 Flink 如何处理状态以及失败的信息，请移步 [流处理系统中的状态]({{ site.baseurl }}/dev/stream/state.html) 页面。
## 给算子赋 ID
**强烈推荐**读者按照本节中的描述进行修改，从而保证你的程序在未来可以顺利升级。主要的区别在于需要通过 uid(String) 方法手动的给算子赋予 ID。这些 ID 将用于确定每一个算子的状态范围。

```
DataStream<String> stream = env.
    // Stateful source (e.g. Kafka) with ID
    .addSource(new StatefulSource())
	    .uid("source-id") // ID for the source operator
		    .shuffle()
	    // Stateful mapper with ID
	    .map(new StatefulMapper())
	    .uid("mapper-id") // ID for the mapper
		    // Stateless printing sink
		    .print(); //Auto-generated ID
			```
			如果不手动给各算子指定 ID，则会由 Flink 自动给每个算子生成一个 ID。只要这些 ID 没有改变就能从保存点（savepoint）将程序恢复回来。而这些自动生成的 ID 依赖于程序的结构，并且对代码的更改是很敏感的。因此，强烈建议用户手动的设置 ID。
### 保存点状态
			可以将保存点（savepoint）想象成一个 算子 ID 到 状态的 Map 结构

			```
			算子 ID     | 状态
			-----------|-------------
			source-id  | 状态源算子的状态
			mapper-id  | 状态化 Mapper 算子的状态
			```
			在上面的例子中，打印输出的 sink 是无状态的，因此不包含在保存点（savepoint）的状态之中。默认，会将每一个保存点状态映射到升级之后的程序中。
## 算子
			用户可以通过 [command line client](https://ci.apache.org/projects/flink/flink-docs-release-1.2/setup/cli.html#savepoints) 来触发保存点（savepoint），取消一个带保存点（savepoint）的作业，从保存点（savepoint）恢复一个作业以及处理一个保存点（savepoint）。
### 触发保存点
			触发保存点（savepoint）的时候，将生成一个包含检查点（checkpoint）元数据的文件。实际的检查点（checkpoint）文件则会保存在用户配置的检查点目录。拿 `FsStateBackend` 或者 `RocksDBStateBackend` 来说:

			```
# Savepoint file contains the checkpoint meta
			/savepoints/savepoint-123123

# Checkpoint directory contains the actual state
			/checkpoints/:jobid/chk-:id/...
			```
			保存点（savepoint）文件通常会比真正的检查点状态要小很多。注意：如果你使用 `MemoryStateBackend`，那么保存点（savepoint）文件将会由自己管理，并且状态也全部有自己管理。
#### 触发一个保存点
			
			```
			$ bin/flink savepoint :jobId [:targetDirectory]
			```
			上面的代码将会为 `:jobid` 触发一个保存点（savepoint）。另外，你还可以指定一个目标路径用于保存保存点（savepoint）文件。这个路径需要给 JobMangager 赋予相应的权限。
			如果没有指定目标路径，你需要有一个 [已经配置好的默认路径](https://ci.apache.org/projects/flink/flink-docs-release-1.2/setup/savepoints.html#configuration)。否则，触发保存点（savepoint）将会失败。
#### 取消带保存点的作业
			
			```
			$ bin/flink cancel -s [:targetDirectory] :jobId
			```
			上面的代码将会自动触发 ID 为 `:jobid` 的作业的一个保存点（savepoint），并且将作业取消掉。另外，你可以指定一个目标路径用于保存保存点（savepoint）文件。这个路径需要给 JobMangager 赋予相应的权限。
			如果没有指定目标路径，你需要有一个 [已经配置好的默认路径](https://ci.apache.org/projects/flink/flink-docs-release-1.2/setup/savepoints.html#configuration)。否则，取消 Job 并触发保存点（savepoint）将会失败。
### 从保存点恢复作业
			
			```
			$ bin/flink run -s :savepointPath [:runArgs]
			```
			上面的语句将提交一个作业，并指定一个保存点（savepoint）路径。作业将从对应的保存点（savepoint）状态进行恢复。保存点（savepoint）文件保存了检查点文件相关的元信息并指向真正的检查点文件。这也是为什么保存点（savepoint）文件通常比检查点文件要小的原因。
#### 跳过从保存点恢复算子
			默认将从保存点（savepoint）文件中进行所有算子的状态恢复。如果你新版的程序中不再有某个算子，那么可以通过 `--allowNonRestoredState` (简写 -n)跳过这些算子。
### 处理保存点
	
			```
			$ bin/flink savepoint -d :savepointPath
			```
	上述命令将会处理掉对应目录的保存点（savepoint）文件。
	一般来说保存点（savepoint）文件会保存在文件系统中，因此用户可以通过操作系统文件来删除保存点（savepoint）。记住保存点（savepoint）仅仅保存了指向检查点数据的元数据。所以，如果你想手动的删除某个保存点（savepoint），你还需要删除检查点文件。由于现在没有直接的方式知道保存点（savepoint）指向那个检查点，因此建议通过系统自带的工具来完成该项工作。
### 配置
	你可以通过设置 `state.savepoints.dir` 来指定默认的保存点（savepoint）文件目标路径。当触发保存点（savepoint）的时候，保存点（savepoint）元数据信息将会保存到该目录中。你可以通过下面的命令来指定一个用户特定的目标路径（查看 [:targetDirectory argument](https://ci.apache.org/projects/flink/flink-docs-release-1.2/setup/savepoints.html#trigger-a-savepoint)获取更多信息）

	```
# Default savepoint target directory
	state.savepoints.dir: hdfs:///flink/savepoint
	```
	如果没有指定一个默认的保存点（savepoint）路径，也没有指定一个用户特定的路径，那么触发保存点（savepoint）将会失败。
## F.A.Q
### 我应该给所有的算子赋予一个 ID 吗
	作为首要原则，当然你应该为每一个算子赋予一个 ID。严格的来说，使用 uid 方法为你作业中所有有状态的算子赋予 ID 就够了。这样的话，保存点（savepoint）将只会包含那些有状态的算子，而不会包含那些无状态的算子。
	在实际使用中，建议给所有的算子赋一个 ID，因为类似 Window 这样的 Flink 内置算子是有状态的，但是并没有显示的说明哪些内置算子是有状态的，哪些是无状态的。如果你很确定某个算子是无状态的，那么可以不给它赋 ID。
### 为什么保存点文件这么小
	保存点（savepoint）文件仅仅包含了相应检查点文件的元信息以及指向检查点文件的指针，而检查点文件通常更大。
	在使用 `MemoryStateBackend` 作为后端存储的情况下，检查点会包含所有的状态，但是被后端限制只保存少量的状态
### 如果我增加了一个带状态的算子会怎么样
	当你在作业中添加了一个算子后，该算子会被初始化为没有保存任何状态。保存点（savepoint）包含所有有状态算子的状态。无状态算子则不在保存点（savepoint）的范围之内。新加入的算子则类似于无状态的算子。
### 如果我删除了一个带状态的算子会怎么样
	默认从保存点（savepoint）恢复的时候，会尝试恢复所有的状态。从一个包含了被删除算子的状态的保存点（savepoint）进行作业恢复将会失败。
	你也可以在运行下面的命令时设置 `--allowNonRestoredState(简称 -n)` 跳过从保存点（savepoint）进行恢复作业:
	
	```
	$ bin/flink run -s :savepointPath -n [:runArgs]
	```
### 如果我对有状态的算进进行重新排序会怎样
	如果你给这些算子赋予了独立的 ID，那么就不影响作业的恢复。
	如果你没有给算子赋予独立的 ID，通常算子进行重排序之后，系统分发的 ID 将会改变，这将会导致从保存点（savepoint）文件恢复失败。
### 如果我增加、删除或者重排序一个无状态的算子会怎样
	如果你给有状态的算子赋予了 ID，那么这些无状态的算子不会影响保存点（savepoint）的恢复。
	如果你没有给有状态的算子赋予 ID，对算子进行重排序之后有状态的算子的自动生成的 ID 会发生变化，这会导致从保存点（savepoint）恢复失败。
### 如果我修改了作业的并发会怎么样
	如果你在 Flink 版本 >= 1.2.0 的系统上触发保存点（savepoint）而且没有使用诸如 `Checkpointed` 等以及过期的 API，你可以从保存点（savepoint）进行恢复并重新设置并行度。
	如果你在低于 1.2.0 的版本上触发保存点（savepoint）或者使用了以及过期的 API。你首先需要将作业升级到 1.2.0 或以上版本。可以参考 [Flink 作业升级指南](https://ci.apache.org/projects/flink/flink-docs-release-1.2/ops/upgrading.html)
