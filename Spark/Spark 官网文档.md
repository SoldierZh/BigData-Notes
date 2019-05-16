# Quick Start

- 在 Spark 2.0 之前，Spark的主要编程接口是 **RDD**， 在 Spark 2.0 之后，**RDD** 被 **DataSet** 取缔。**DataSet** 是一种强类型的 **RDD**，支持**RDD**所有接口的同时，进行了更多的优化，**DataSet**的性能要比**RDD**好，所以官网推荐使用 **DataSet**。**直接使用SparkSession，而不要使用SparkContext。** 
- Spark 的交互式命令行只支持 Python 和 Scala 两种，以下只例举 Scala

---

通过如下命令进入 spark shell：

```shell
./bin/spark-shell
```

**DataSet** 是Spark最主要的抽象：一种分布式的项目（这里可以理解为多条数据记录）集合。

```scala
scala> val textFile = spark.read.textFile("README.md")
textFile: org.apache.spark.sql.Dataset[String] = [value: string]
```

可以通过一系列 **transform** 和 **action** 函数操作 **DataSet**，也就是 **DataSet** 提供的一些接口，例如 `map`, `filter`, `collect`…. 如下，可以统计单词数：

```scala
scala> val wordCounts = textFile.flatMap(line => line.split(" ")).groupByKey(identity).count()
wordCounts: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count(1): bigint]
```

可以通过 `cache()` 将 **DataSet** 缓存到集群的内存缓存中，之后如果再使用已经缓存了的 **DataSet**，将不会再次对其进行计算，直接可以使用。

```scala
scala> linesWithSpark.cache()
res7: linesWithSpark.type = [value: string]

scala> linesWithSpark.count()
res8: Long = 15
```

---

可以编写Scala程序，并生成jar包，通过 `spark-submit` 提交到Spark执行：

```scala
import org.apache.spark.sql.SparkSession

object SimpleApp {
  def main(args: Array[String]) {
    val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
    val spark = SparkSession.builder.appName("Simple Application").getOrCreate()
    val logData = spark.read.textFile(logFile).cache()
    val numAs = logData.filter(line => line.contains("a")).count()
    val numBs = logData.filter(line => line.contains("b")).count()
    println(s"Lines with a: $numAs, Lines with b: $numBs")
    spark.stop()
  }
}
```

1. 需要一个 `object` 以及 `main`函数，类似Java中的 `static main`
2. 获得 SparkSession 实例，这个实例就相当于 spark-shell 中的 `spark`对象
3. 编写业务逻辑代码
4. 通过 maven或者 sbt 对scala项目代码进行编译并打包
5. 通过 `spark-submit`提交应用程序

# RDD Programming

- Spark提供了两种类型的算子
  - **transformation**： lazy机制
  - **action**：触发真正的计算
- 在Spark中，一个分区对应一个task,

- 造成**shuffle**的算子包括：

  - 重分区相关：`repartition`,  `coalesce`
  - ByKey相关：`groupByKey`, `reduceByKey`
  - join相关：`cogroup`, `join`

- Spark 提供了两种共享变量机制

  - **broadcast variable**: 在每台节点上缓存的**只读**变量，而不是随着task拷贝到节点的副本。在创建的时候，一次性将变量广播到所有节点，之后使用时，不需要随着task拷贝到节点。

  ```scala
  scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))  // 将变量一次性广播到所有节点上
  broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)
  
  scala> broadcastVar.value
  res0: Array[Int] = Array(1, 2, 3)
  ```

  - **accumulator**：一个只能增加的变量，可用于全局的 counter 或者 sum。**Note**: executor可以通过`add`增加 accumulator 的值，但只有 driver 可以通过 `value` 读取 accumulator 的值。除了内建的 Long、Double等类型的 accumulator， 可以自己实现自定义类型的 accumulator。

  ```scala
  scala> val accum = sc.longAccumulator("My Accumulator")
  accum: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 0, name: Some(My Accumulator), value: 0)
  
  scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
  ...
  10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s
  
  scala> accum.value
  res2: Long = 10
  ```

  accumulator 的add操作依然满足rdd的lazy机制：

  ```scala
  val accum = sc.longAccumulator
  data.map { x => accum.add(x); x }
  // Here, accum is still 0 because no actions have caused the map operation to be computed.
  ```

# Cluster Mode Overwiew

编程模型：

- **Application**: 用户编写的Spark应用程序，包含main函数，创建的 **SparkContext** 对象，以及在RDD上的一系列 `transformation` 和 `action`算子组成;

- **Job**：由 `action` 算子产生，一个 **Application** 会包含一个或多个 **Job**;

- **Stage**：每个 **Job** 会根据 shuffle 划分为多个 **Stage**，每个**Stage**会对应多个可并行计算的**Task**（计算过程一样，输入不一样，每个分区对应一个 Task）。两个 **Stage ** 之间相当于 MapReduce，上游 Stage 相当于 Mapper， 下游 Stage 相当于 Reducer，上游 Stage 结束之后需要 shuffle write，下游 Stage 需要 shuffle read，与 MapReduce 操作类似。
- **Task**：计算任务单元，每个分区会对应一个 **Task** ，Task 会被 Task调度器发送到对应的 Executor， 每个 Executor 可以接受多个 Task。

架构模型：

- **Cluster Manager**： 集群的资源管理器， 可以是 Yarn， Mesos等外部资源管理器；
- **Driver** ：运行 application main 函数的进程，在这个进程中会创建 SparkContext 对象。 此进程可以运行在客户端 `--deploy-mode client`，或者 Worker中 `--deploy-mode cluster`。如果 `--master yarn` 并且 `--deploy-mode cluster`，则Driver会运行在Yarn的 **ApplicationMaster** 中。
- **Executor**：在 Worker 中运行的进程，负责运行 task 并且将数据缓存到内存或者磁盘中。

- **Worker**：从节点，相当于 Yarn 集群中的 **NodeManager**。

- **Master**：主节点，相当于 Yarn 集群中的 **ResourceManager**。

  

# Running Spark on Yarn

运行命令：

- `spark-submit`： 提交编写打包好的 Application jar 到集群执行

- `spark-shell`:  进入Spark shell 的交互界面，使用 Scala 语言

- `spark-sql`： 进入 Spark Sql 的交互界面，使用SQL

主要参数说明：

- `--master yarn` : 运行在 Spark 集群上
- `--deploy-mode client`， driver 运行在客户端进程中，也就是执行 spark-submit ，spark-shell 或者 spark-sql 命令的机器进程中 ，一般 spark-sql 和 spark-shell 的模式最好为 client 
- `--deploy-mode cluster`， driver 运行在 ApplicationMaster 进程中
- `--queue myqueue` 指定YARN的队列名称
- `--jars my-other-jar.jar,my-other-other-jar.jar` 添加其他jar包
- `--files ` 添加文件

**Note**：Yarn 相关的参数使用 `spark.yarn.`前缀

# Spark Configuration

Spark 提供了三个地方可以配置系统

## Spark Properties

Spark Properties 控制了大部分的属性配置，每个Application的配置是独立的，Application之间相互不影响。

Spark Properties 的配置方式：

- 方式一：可以直接通过 **SparkConf**配置， 但这是一种硬编码的方式:  （优先级最高）

- ```scala
  val conf = new SparkConf()
               .setMaster("local[2]")
               .setAppName("CountingSheep")
  val sc = new SparkContext(conf)
  ```

- 方式二：可以通过动态加载的方式

  ```shell
  ./bin/spark-submit --name "My app" --master local[4] --conf spark.eventLog.enabled=false
    --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
  ```

- 方式三：通过 `conf/spark-defaults.conf` 文件进行配置

**Note**： Spark 会按照方式 三二一 的顺序加载配置参数，后加载的会覆盖之前加载的参数，所以方式一配置的参数优先级最高。

可以通过 Spark Web UI 的 **Environment** tab页查看设置的属性，只有通过以上三种方式配置的参数才会显示，默认参数不会显示。 

## Environment Variables

通过在 `spark-env.sh` 中设置

## Logging

# Spark 调优

由于Spark大部分计算是基于内存的特性，Spark程序的瓶颈会受到集群中很多资源的影响： CPU，网络，内存。

## 数据序列化

有两种序列化库：

- Java serialization：Java自带的序列化方法，序列化很慢，而且会序列化很多不必要的数据，导致序列化之后的数据量很大；
- Kryo serialization: 序列化速度很快，并且序列化更加紧凑（是Java serialization的是10倍） ，但不支持所有的 Serialization 类型，需要提前将Class 注册到**SparkConf**中。 

```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
val sc = new SparkContext(conf)
```

## 内存调优

内存调优有三个注意事项：对象占用的空间，访问这些对象成本，垃圾收集的成本。

默认情况下，Java对象的访问速度很快，但是要比序列化之后对象的多占用2-5倍的空间。

### 内存管理概述

Spark 中的内存使用大致属于以下两种类别之一：**execution** 和 **storage**。

- **execution内存**：用于shuffles，joins，sorts 和 aggregates等计算的内存；
- **storage内存**：用于缓存和传播内部数据的内存。

**execution** 和 **storage** 共享一个统一的内存（M），当 **execution** 没有占用内存时，**storage**可以获得所有可用内存，反之亦然。如有必要，**execution** 可以抢占 **storage** 的内存，但会低于一个阈值（R），也就就是保证了最小**storage**空（R），这部分的**storage**空间不会被抢占，**execution**最多使用 M-R 的内存。反过来，**storage** 不可以抢占**execution**的内存。

这么设计的好处：

- 对于不需要缓存的application，**execution** 可以使用整个内存（M-R），避免了不必要的溢写。
- 需要缓存的application可以保留最小的**storage** (R)，这部分**storage**内存的数据不会被清除。
- 用户无需知道如何具体划分内存，就可以直接使用，而无需担心性能。

有两个相关的配置项，一般用户不需要调整它们：

- `spark.memory.fraction` : (JVM堆内存 - 300M) * fraction ， 默认值是0.6，这部分内存即为 **execution** 和 **storage** 共享的内存（M）。剩下的40%的内存用于存储用户数据结构，Spark中的内部元数据，以及在异常大的记录情况下防止OOM异常。
- `spark.memory.storageFraction`:  R = M * storageFraction， 默认值为0.5，这部分内存不会被**execution**抢占。

### 确定内存消耗

调整数据集所需内存消耗量的最佳方法是创建一个RDD，将其放在缓存中，然后查看Web UI中的 **Storage** 页面，就可以获知这个RDD占用了多少内存。

要估计特定对象的内存消耗，需要使用 `SizeEstimator.estimate()` 方法，这对于尝试使用不同的数据布局来调整内存使用情况，以及确定广播变量在每个 executor 堆中占用的空间量非常有用。

### 数据结构调优

减少内存消耗的第一种方法是避免会增加开销的Java功能，例如基于指针的数据结构和包装类对象，具体可以如下操作：

- 自定义数据结构时尽量使用对象数组和基本类型，而不要使用Java或者Scala集合类和包装类型。
- 尽量避免使用包含大量小对象和指针的嵌套结构；
- 考虑使用 数值型的ID或者枚举对象，而不要使用 String 类型的 key

### 序列化RDD

如果经过数据结构优化，对象依然很大而无法有效存储，这时候就要考虑序列化RDD对象，可以通过 RDD 持久化 API中的 `StorageLevels` 的 `MEMORY_ONLY_SER` 方法序列化RDD对象，Spark会将每个RDD分区存储为一个大字节数组。由于需要动态反序列化每个对象，因此以序列化形式存储没数据的唯一缺点是访问对象的时间较慢。**推荐使用 Kyro 序列化方式**。

### GC调优

垃圾回收的成本与Java对象的**数量**成正比，使用Array取代ArrayList会降低垃圾回收的成本，或者可以将Java对象序列化之后存储在内存中，这样每个RDD的分区就会对应一个大的字节数组对象。**在尝试其他技术对GC进行调优之前，先尝试序列化缓存的方式**。

#### 评估GC的影响

GC调优的第一步是收集有关**GC的发生频率**和**GC使用时间**的统计信息，这可以通过添加 `-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps`到Java options中来实现，[configuration](https://spark.apache.org/docs/latest/configuration.html#runtime-environment) 中描述了在Spark中配置Java参数。下次运行Spark作业时，每次发生GC时，相关信息会被打印到 Worker 的 log中。**Note**：这些log将位于集群的Worker节点上（位于其工作目录中的stdout文件中），而不是位于 Driver 节点上。

#### 高级GC调优

为了进一步GC调优，需要先了解JVM相关内存管理信息：

- Java 堆空间被划分为 Young 和 Old 两个区域，Young 区域存储生命周期较短的对象，Old区域存储生命周期较长的对象；
- Young区域进一步被划分为三个部分： Eden， Survivor1， Survivor2；
- GC过程：当 Eden 区域满了之后，会在Eden区域启动一次 minor GC，并将 Eden 和 Survivor1中存活的对象复制到 Survivor2 中。Survivor区域会被交换（之前的Survivor1会变成新的Survivor2，之前的 Survivor2会变成新的 Survivor1 ）。如果一个对象足够大或者 Survivor2 满了，则将存活的对象移到 Old 区域。最后，当Old接近满了的时候，将会启动一次 Full GC。 **（Eden + Survivor1 可以共同为短期对象提供存储空间，Survivor2 作为Survivor 区域用来为 Minor GC 提供存活对象的存储空间，所以Young区域如果总共有4份，则实际可存储空间为3份）**

Spark中的GC调优目标是确保只有长期存在的RDD对象会被存储在Old区域，并且 Young 区域有足够大小存储短期对象。这将有助于避免 Full GC 收集在 task 执行期间创建的临时对象。以下是一些有用的步骤：

- 通过收集 GC 统计数据来检查是否有太多GC，如果在一个task完成之前发生了多次 **Full GC**，这就意味着没有足够的内存用于执行任务；
- 如果有多次 **Minor GC**，而不是多次 **Full GC**， 则需要为 Eden 区域分配更多空间，可以将Eden的大小设置为超过每个task所需要的预计内存大小。**如果确定Eden的大小为E, 则可以使用参数： -Xmn = 4/3 * E 设置Young 代的大小** （只能通过设置 Young 的大小，间接设置 Eden 区域大小）
- 在打印的GC统计信息中，如果 Old 区域接近满，则通过 `spark.memory.fraction` 来减少用于缓存的内存量，缓存更少的对象比减慢task执行速度更好。或者，可以考虑减少 Young 的内存大小 （-Xmn）。
- 尝试使用 `-XX:+UseG1GC`来使用 G1 垃圾收集器，在GC平静瓶颈的某些情况，G1 可以提高性能。对于较大的 executor 堆内存，可以通过 `-XX:G1HeapRegionSize`增加 G1 region 的大小。
- 如果 task 是从 HDFS 读取数据，则可以使用从 HDFS 读取的数据块大小来估计任务使用的内存量。解压缩block的大小通常是block大小的3倍，所以，如果我们希望有4个task的工作空间，并且HDFS的block大小为128M，则可以估计 Eden 的大小为 4 X 3 X 128M 。

**Note**：控制 Full GC 的频率有助于减小开销。

## 其他调优

### 并行度

Spark 会根据每个文件大小自动设置 Map Tasks 的数量，当然也可以通过 `SparkContext.textFile`的可选参数来控制这个数量。

对于分布式的 Reduce 算子，比如 `groupByKey` 和`reduceByKey`，它会使用最大的父RDD的分区数。也可以通过[`spark.PairRDDFunctions`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions) 的第二个参数设置并行度，或者通过配置 `spark.default.parallelism` 改变默认值。一般，集群中每个 CPU core 有2-3个 Task。

### Reduce Tasks的内存使用

有时候，出现 OOM 异常不是因为RDD太大不能存储在内存中，而是因为一个task的工作集太大导致的，例如 `groupByKey`中的 reduce 任务太大。Spark 的 shuffle操作（sortByKey，groupByKey，reduceByKey,  join等）在每个 task 中构建一个 Hash 表来执行分组，这通常很大。**最简单的解决方法是增加并行度，以便每个task的输入集要更小**。Spark 可以有效地支持少于200ms的短task，因为它在许多task中重用了一个 executor JVM，并且它具有较低的 task 启动成本，因此可以安全的将并行度提高到超过集群中的核心数。

### 将大变量变成广播变量

使用 SparkContext 提供的广播变量功能，可以大大减少每个序列化任务的大小，以及在集群中启动job的成本。如果 tasks 需要使用任何 driver 程序中的大对象，则最好将这个大对象转换为 广播变量。**Spark会在Master上打印每个task的序列化大小，因此可以查看它以确定task是否过大，大约20KB的task可能需要优化。**

### 数据本地化

可以通过 `spark.locality` 相关配置参数来调整数据本地化。

#  Job Scheduling

### 动态资源分配

需要配置两个参数来开启动态资源分配功能：

- `spark.dynamicAllocation.enabled=true`
- `spark.shuffle.service.enabled=true`

开启外部 shuffle 服务的目的是允许删除 executor， 但是不删除executor产生的 shuffle 文件。

### Application中的调度

**一个 Application 中，在不同线程中提交的 Job 会并发执行。** 这里的Job是指 Spark 的 action算子产生的 Job。Spark 的调度器是线程安全的。

默认情况下，Spark的调度器是以**FIFO**方式运行Job的。每个Job被划分为多个 Stage(map和reduce阶段)， 第一个 Job 对于所有资源具有最高优先权，然后是第二个Job，以此类推。如果队列头部的Job不需要使用整个集群资源，则之后的Job可以理解开始运行，但是如果队列头部的Job很大，则后续Job可能会显著延迟。

从Spark 0.8 开始，可以配置 **Fair Scheduler** 方式调度 Job。Spark 会以循环的方式在 Job之间分配 tasks，以便所有 Job 获得大致相等的 集群资源份额。这意味着在一个长作业执行期间提交的短作业可以获得资源，并获得良好的响应时间，而无需等待长作业执行结束。

```
 spark.scheduler.mode=FAIR
```

### Fair Scheduler Pools

类似Yarn的Scheduler，可以有多个队列，每个队列占有不用比例的集群资源，以及不同的优先级，Job可以被提交到不同的队列中，在每个队列内部采用 FIFO 方式调度Job。

# Spark SQL

Spark SQL 是用于结构化数据处理的Spark模块。

- **DataSet** 综合了RDD的特性（强类型，功能强大的lambda函数）以及Spark SQL的优化执行引擎，所以性能更好。推荐使用 DataSet 替换 RDD。**DataSet与RDD类似，但是DataSet不使用Java序列化方法或Kryo，而是使用专用的 Encoder 来序列化对象。Encoder是动态生成的代码，它使用一种格式，允许Spark执行很多操作，例如：过滤，排序，和hash，而无需将字节数组反序列化。**
- **DataFrame** 是一种按照列组织的 **DataSet**。在概念上，它等同于关系型数据库的Table，或则会 Python 中的 data frame，但在底层具有更多的优化。**DataFrame** 可以通过各种来源构建，例如：结构化数据，Hive中的表，外部数据库或者现有的 RDD。**在 Scala 中，DataFrame 相当于 DataSet[Row]。** `case class Person(name: String, age: Int)`  `DataSet[Person]` 相当于 `DataSet[Row[String, Int]]`。

## Getting Start

**SparkSession**： 是Spark所有功能的入口点，可通过以下代码简单构建一个 SparkSession对象：

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder()
  .appName("Spark SQL basic example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate()

// For implicit conversions like converting RDDs to DataFrames
import spark.implicits._

```

通过SparkSession对象创建 DataFrame

```scala
val df = spark.read.json("examples/src/main/resources/people.json")
// Displays the content of the DataFrame to stdout
df.show()
```

- Spark 会为 case class ，基本类型，自动创建 Encoder 

```scala
case class Person(name: String, age: Long)

// Encoders are created for case classes
val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show()
// +----+---+
// |name|age|
// +----+---+
// |Andy| 32|
// +----+---+

// Encoders for most common types are automatically provided by importing spark.implicits._
val primitiveDS = Seq(1, 2, 3).toDS()
primitiveDS.map(_ + 1).collect() // Returns: Array(2, 3, 4)

// DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name
val path = "examples/src/main/resources/people.json"
val peopleDS = spark.read.json(path).as[Person]
peopleDS.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+
```

## DataSet 与 RDD 的交互

