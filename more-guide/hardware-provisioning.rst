硬件配置
==========================================

Spark 开发者们常常被问到的一个问题就是：如何为 Spark 配置硬件。但具体的硬件配置还依赖于实际的使用情况, 我们通常会给出以下的建议。


存储系统
------------------------------------------

因为绝大多数Spark作业都很可能是从外部存储系统加载输入数据（如：HDFS 或者 HBase），所以最好把 Spark 部署在离这些存储比较近的地方。建议如下：

* 只要有可能，就尽量在 HDFS 相同的节点上部署 Spark。最简单的方式就是，在 HDFS 相同的节点上独立部署 Spark（standalone mode cluster），并配置好 Spark 和 Hadoop 的内存和 CPU 占用，以避免互相干扰（对 Hadoop 来说，相关的选项有 mapred.child.java.opts – 配置单个任务的内存，mapred.tasktracker.map.tasks.maximun和mapred.tasktracker.reduce.tasks.maximum – 配置任务个数）。当然，你也可以在一些通用的集群管理器上同时运行 Hadoop 和 Spark，如：Mesos 或 Hadoop YARN。
* 如果不能将 Spark 和 HDFS 放在一起，那么至少要将它们部署到同一局域网的节点中。
* 对于像 HBase 这类低延迟数据存储来说，比起一味地避免存储系统的互相干扰，更需要关注的是将计算分布到不同节点上去。


本地磁盘
------------------------------------------

虽然大部分情况下，Spark 都是在内存里做计算，但它仍会使用本地磁盘存储数据，如：存储无法装载进内存的 RDD 数据，存储各个阶段（stage）输出的临时文件。因此，我们建议每个节点上用4~8块磁盘，非磁盘阵列方式挂载（只需分开使用单独挂载点即可）。在 Linux 中，挂载磁盘时使用 noatime option 可以减少不必要的写操作。在Spark中，配置（configure）spark.local.dir 属性可指定Spark使用的本地磁盘目录，其值可以是逗号分隔的列表以指定多个磁盘目录。如果该节点上也有 HDFS 目录，可以和 HDFS 共用同一个块磁盘。


内存
------------------------------------------

一般来说，Spark 可以在8GB~几百GB内存的机器上运行得很好。不过，我们还是建议最多给 Spark 分配75%的内存，剩下的内存留给操作系统和系统缓存。

每次计算具体需要多少内存，取决于你的应用程序。如需评估你的应用程序在使用某个数据集时会占用多少内存，可以尝试先加载一部分数据集，然后在 Spark 的监控UI（http://<driver-node>:4040）上查看其占用内存大小。需要注意的是，内存占用很大程度受存储级别和序列化格式影响 – 更多内存优化建议，请参考调优指南（tuning guide）。

最后，还需要注意的是，Java 虚拟机在 200GB 以上内存的机器上并非总是表现良好。如果你的单机内存大于200GB，建议在单个节点上启动多个 worker JVM。在Spark独立部署模式下（standalone mode），你可在conf/spark-env.sh 中设置 SPARK_WORKER_INSTANCES 来配置单节点上worker个数，而且在该文件中你还可以通过 SPARK_WORKER_CORES 设置单个worker占用的CPU core个数。


网络
------------------------------------------

以我们的经验来说，如果数据能加载进内存，那么多数Spark应用的瓶颈都是网络带宽。对这类应用，使用万兆网（10 Gigabit）或者更强的网络是最好的优化方式。对于一些包含有分布式归约相关算子（distributed reduce相关算子，如：group-by系列，reduce-by系列以及SQL join系列）的应用尤其是如此。对于任何一个应用，你可以在监控UI (http://<driver-node>:4040) 上查看Spark混洗跨网络传输了多少数据量。


CPU Cores
------------------------------------------

Spark 在单机几十个 CPU 的机器上也能表现良好，因为 Spark 尽量减少了线程间共享的数据。但一般你至少需要单机8~16个CPU cores。当然，根据具体的计算量你可能需要更多的CPU，但是：一旦数据加载进内存，绝大多数应用的瓶颈要么是 CPU，要么是网络。