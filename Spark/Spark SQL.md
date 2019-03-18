# 1. Spark 基础知识

## 1.1. RDD 编程模型

> RDD是直接在编程接口层面提供了一种**高度受限的共享内存模型**。
>
> RDD是Spark的核心数据结构，全称是**弹性分布式数据集（Resilient Distributed Dataset）**，其本质是一种分布式的内存抽象，表示一个 **只读的数据分区（Partition）集合** 。

> RDD之间的**依赖（Dependency）**关系包含两种：
>
> - **窄依赖**：RDD之间的分区是一一对应的；
> - **宽依赖**：下游RDD的每个分区与上游RDD（也称为父RDD）的每个分区都有关，是多对多的关系。
>
> 对于窄依赖，数据可以通过类似管道（Pipeline）的方式全部执行；
>
> 对于宽依赖，数据需要在不同节点之间Shuffle传输。

>RDD计算的时候会通过一个`compute`函数得到每个分区的数据：
>
>- 若 RDD 是通过已有文件系统构建，则`compute`函数读取指定文件系统中的数据；
>- 若 RDD 是通过其他RDD转换而来，则`compute`函数执行逻辑转换，将其他RDD数据进行转换。

> RDD的 **操作算子** 包含两类：
>
> - `transformation`，用来将RDD进行转换，构建RDD的依赖关系；
> - `action`，用来触发RDD的计算，得到RDD的相关计算结果或者将RDD保存到文件系统中。

> **总结**: 基于 RDD 的计算任务可描述为：从稳定的物理存储（如分布式文件系统HDFS）中加载记录，记录被传入由一组确定性操作构成 的 DAG (有向无环图) ，然后写回稳定存储。

> **容错性**：在实际执行中，RDD通过Lineage信息来完成容错，即使出现数据分区丢失，也可以通过 Lineage 信息重建分区。

```scala
// word count
def main(args: Array[String]) {
    val conf = new SparkConf()
    val sc = new SparkContext(conf)
    val result = sc.textFile("hdfs://...")
    	.flatMap(line => line.split(" "))
    	.map(word => (word, 1))
    	.reduceByKey(_ + _)
    result.collect()			// 真正触发执行
}
```

## 1.2.DataFrame 和 DataSet

> **DataFrame** 与 RDD:
>
> - 相同点：都是不可变的分布式弹性数据集
> - 不同点：RDD 中的数据不包含任何结构信息，直接使用 RDD 时需要开发人员实现特定的函数来完成数据结构的解析；DataFrame 中的数据集类似于关系数据库中的表，按列名存储，具有Schema信息，开发人员可以直接将结构化数据集导入DataFrame。

```scala
val lineDF = sc.textFile("hdfs://...").toDF("line")
val wordDF = lineDF.explode("line", "word")((line: String) => line.split(" "))
val wordCountDF = wordDF.groupBy("word").count()
wordCountDF.collect()
```

>**DataSet** 和 DataFrame：
>
>DataFrame 本质上是一种特殊的 DataSet（DataSet[Row]类型），DataSet 是对 DataFrame 的扩展。
>
>DataSet 具有两种完全不同的 API 特征：
>
>- 强类型（Strongly-Typed）：一般通过Scala中定义的 Case Class 或者 Java中的 Class 指定
>- 弱类型（Untyped）
>
>DataSet 结合了 RDD 和 DataFrame 的优点，提供**类型安全**和**面向对象**的编程接口，并引入了**编码器 (Encoder) **的概念。

```scala
case class Person(name: String, age: Long)  // 起到了 Encoder 的作用
val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show()
```

> Encoder 不仅能够在编译阶段完成类型安全检查，还能够生成字节码与堆外数据进行交互，提供对各个属性的按需访问，而不必对整个对象进行反序列化操作，大大减少了网络数据传输的代价。

# 2. Spark SQL 执行全过程概述

## 2.1 从 SQL 到 RDD 概述

> 从SQL到 Spark 中 RDD 的执行需要经过两个阶段：
>
> - 逻辑计划（LogicalPlan）：Unresolved LogicalPlan  => Analyzed LogicalPlan => Optimized LogicalPlan
> - 物理计划（PhysicalPlan）：Iterator[PhysicalPlan] => SparkPlan => Prepared SparkPlan
>
> 物理计划的最后阶段会执行action操作，即可提交执行。

> **从SQL语句的解析一直到提交之前，上述整个转换过程都在Spark集群的Driver端进行，不涉及分布式环境。** 
>
> `SparkSession.sql(sqlStr: String)` 调用SessionState中的各种对象，包括上述不同阶段对应的 SparkSqlParser类、Analyzer类、Optimizer 类 和 SparkPlanner 类等，最后封装成一个 `QueryExecution`对象。

```json
student.json
{"id": 1, "name":"zhang", "age":29}
{"id": 2, "name":"qian", "age":20}
{"id": 3, "name":"xu", "age":11}
{"id": 4, "name":"li", "age":21}
{"id": 5, "name":"wang", "age":32}
```

```scala
val spark = SparkSession.builder().appName("example").master("local").getOrCreate() 
spark.read.json("student.json").createOrReplaceTempView("student") // 本质上是 SQL 的 DDL
spark.sql("select name from student where age > 18").show()
```

> 示例中 sql 语句（不包含 Join 和 Aggregate 操作）：
>
> - 生成的 LogicalPlan 中包含三个节点：Relation(对应数据表student)、Filter(对应过滤逻辑 age > 18)、Project(对应列裁剪，只涉及3列中的2列)；

```
Project => Filter => Relation
```

> - 生成的 PhysicalPlan ，由 LogicalPlan 一一映射得到：Relation 节点转换为 FileSourceScanExec 执行节点，Fileter 节点转换为 FilterExec执行节点，Project 节点转换为 ProjectExec 执行节点。

```
ProjectExec => FilterExec => FileSourceScanExec
```

> 树的根节点是 ProjectExec，每个节点都`execute` 函数，将从根节点开始递归调用算子树中的每个节点的 `execute` 函数，也就是从叶子结点开始执行，实际转换为对RDD的操作：

```scala
val rdd0 = FileSourceScanExec.inputRDD
val rdd1 = rdd0.FileSoureScanExec_execute()
val rdd2 = rdd1.FilterExec_execute()
val rdd3 = rdd2.ProjectExec_execute()
```

## 2.2 重要概念

> **Catalyst**: 是Spark SQL 内部实现上述流程中平台无关部分的基础架构。以下介绍 Catalyst 中涉及的重要概念和数据结构。

### 2.2.1 InternalRow 体系

> **InternalRow** 表示一行行数据的类，InternalRow 每一个元素的类型都是Catalyst内部定义的数据类型， PysicalPlan 转换的RDD实质是 RDD[InternalRow]。

### 2.2.2 TreeNode 体系

### 2.2.3 Expression 体系

## 3. 内部数据类型体系





