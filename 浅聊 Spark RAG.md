## 前置知识：
### RDD 与算子：
#### RDD（类似 Stream）
RDD（弹性分布式数据集）是 Spark 的核心抽象，代表一个分布式的、不可变的数据集合。它允许开发人员以容错的方式在大规模数据集上进行并行计算。
**特点：**
- **不可变性**：一旦创建，RDD 就不能更改。你可以通过转换（如 map 或 filter）来生成新的 RDD。
- **分布式**：RDD 会自动将数据分区并分布在集群的多个节点上。
- **容错性**：RDD 可以自动从节点失败中恢复，利用数据的血统（lineage）信息来重新计算丢失的分区。
创建 RDD 的两种主要方式：
1. 从数据源创建：比如从 HDFS、S3、HBase 等外部存储系统中读取数据。
2. 从集合并行化创建：将一个本地集合（如 List）并行化成一个 RDD。
#### 算子
Spark 中的 **算子** 是对 RDD 进行操作的函数，分为 **转换算子（Transformations）** 和 **行动算子（Actions）**。
1. **转换算子（Transformations）**
- **定义**：对 RDD 进行转换操作时，不会立即执行计算，而是生成一个新的 RDD，并记录转换的操作链。当有行动算子调用时，整个操作链才会被执行。
- **常见转换算子**：
  - `map`：对 RDD 中的每个元素进行处理，并返回一个新的 RDD。
  - `filter`：对 RDD 中的每个元素进行过滤，保留满足条件的元素。
  - `flatMap`：类似 map，但可以将每个元素映射为多个输出元素，返回一个扁平化的 RDD。
  - `reduceByKey`：用于键值对 RDD，将具有相同键的值合并。
  - `groupByKey`：将具有相同键的值分组。
  - `join`：连接两个键值对 RDD，类似于 SQL 中的连接操作。
2. **行动算子（Actions）**
- **定义**：行动算子会触发实际的计算，并将结果返回给驱动程序或将数据写入外部存储。
- **常见行动算子**：
  - `collect`：将 RDD 中的数据收集到一个数组中并返回到驱动程序（注意，大数据集不推荐使用）。
  - `count`：计算 RDD 中的元素数量。
  - `first`：返回 RDD 中的第一个元素。
  - `take`：返回 RDD 中的前 n 个元素。
  - `saveAsTextFile`：将 RDD 的数据保存到一个文本文件中。
  - `reduce`：对 RDD 中的所有元素进行归约操作，并返回单一结果
## DAG 是什么
DAG （Directed Acyclic Graph）：在 Spark 中，DAG（有向无环图）是一种数据结构，它描述了一系列的计算操作如何相互依赖。图中的每个节点代表一个 RDD(类似 Java 中的 Stream) 的转换操作（如 map、filter），而边表示操作之间的数据依赖关系。
简图如下:
![image.png](https://upload-images.jianshu.io/upload_images/11859806-81e28a3cfe16f97b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
类似于工厂的流水线，其中每个节点表示加工步骤，而边表示每个步骤的依赖。只有在所有前置加工步骤完成后，流水线才能继续进行后续的步骤
## 为什么 Spark 使用 DAG？
1.  **优化执行**：Spark 使用 DAG 来组织和优化任务调度，而不是每个操作都单独执行。这样可以合并多个转换操作，并减少不必要的中间数据传输。
2.  **减少 I/O 开销**：通过构建一个全局的计算计划，DAG 可以有效地减少数据写入磁盘和从磁盘读取的开销，从而提升整体性能。假设你有一个场景，原始逻辑是将中间结果保存到磁盘并再次读取，如:
```scala
val data = sc.textFile("input.txt") 

val processedData = data.map(line => line.toUpperCase()) 

processedData.saveAsTextFile("intermediate_output")  

val reloadedData = sc.textFile("intermediate_output") 

val finalResult = reloadedData.filter(line => line.startsWith("A")) 

finalResult.saveAsTextFile("final_output")
```
使用 DAG，Spark 会优化执行流程，避免这些不必要的磁盘操作。它会将 map 和 filter 操作组合在一起，并且只在 saveAsTextFile 处才进行一次数据写入操作，从而减少磁盘 I/O。
## 生成 DAG
### 生成 RAG
用户写的一堆 RDD（类似Stream） 操作
```scala
val data = sc.textFile("input.txt") 
val words = data.flatMap(_.split(" ")) 
val wordPairs = words.map(word => (word, 1)) 
val wordCounts = wordPairs.reduceByKey(_ + _) 
wordCounts.saveAsTextFile("output")
```
然后 Spark 将这些 RDD 操作转换成一个 DAG 如下：
![image.png](https://upload-images.jianshu.io/upload_images/11859806-a92c08deda7936e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 划分阶段（Stages）
每个 RDD 都有以下五个核心属性
![image.png](https://upload-images.jianshu.io/upload_images/11859806-3bb28520639fa031.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
例图:![image.png](https://upload-images.jianshu.io/upload_images/11859806-207dbcdd9590de85.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中依赖分为窄依赖和宽依赖：
![image.png](https://upload-images.jianshu.io/upload_images/11859806-efc5bf93b7b973ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
回到正题：划分阶段（Stage）的主要判断依据就是当前 RDD 的依赖是否是宽依赖，如果是宽依赖则会生成一个新的 Stage ，因为当前 RDD 需要等待宽依赖 shuffle（洗牌） 完才能操作
比如上面的代码就可以划分为两个阶段：
![image.png](https://upload-images.jianshu.io/upload_images/11859806-5ecdc1c0ae11b2e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---------------------------------后续的重点---------------------------
### 生成任务
每个阶段生成一个任务集（task set） , 任务集里的每个任务会分发到不同的机器上处理 rdd 里的一个分区
### 失败容错
如果某个任务执行失败了，会按照 DAG 去查询当前任务的分区依赖的分区，并只恢复依赖的分区，无需重新执行整个作业

# 引用
https://www.infoq.cn/article/lbzkjpoafare5c0ci4ur
https://blog.csdn.net/m0_49834705/article/details/113111596
