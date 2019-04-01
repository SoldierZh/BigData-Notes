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
Project --> Filter --> Relation
```

> - 生成的 PhysicalPlan ，由 LogicalPlan 一一映射得到：Relation 节点转换为 FileSourceScanExec 执行节点，Fileter 节点转换为 FilterExec执行节点，Project 节点转换为 ProjectExec 执行节点。

```
ProjectExec --> FilterExec --> FileSourceScanExec
```

> - 生成对RDD的操作，树的根节点是 ProjectExec，每个节点都有`execute` 函数，将从根节点开始递归调用算子树中的每个节点的 `execute` 函数，也就是从叶子结点开始执行，实际转换为对RDD的操作：

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
>
> InternalRow 是根据**下标**来访问和操作列元素的。
>
> InternalRow 是一个抽象类，其子类有：
>
> - **BaseGenericInternalRow** 抽象类
>   - **GenericInternalRow**：采用 `Array[Any]` 进行底层存储，不允许使用 `set`方法对元素进行修改；
>   - **SpecificInternalRow**：采用 `Array[MutableValue]` 进行底层数据存储，可以使用 `set`方法对元素进行修改
>   - **MutableUnsafeRow**
> - **JoinedRow**：用于Join操作，将两个 InternalRow 放在一起形成新的 InternalRow。**使用时需要注意构造参数的顺序**。
> - **UnsafeRow**：不采用 Java 对象存储的方式，避免了 JVM 中垃圾回收的代价。此外， UnsafeRow 对行数据进行了特定的编码，使得存储更加高效。 

### 2.2.2 TreeNode 体系

> **TreeNode** 是 Spark SQL 中所有树结构（**LogicalPlan**，如`Filter`、`Project`等类，**PhysicalPlan**，如 `FilterExec`、`ProjectExec`等类）的基类，定义了一系列通用的集合操作和树遍历操作接口。
>
> TreeNode 一直在内存中维护，不会 dump 到磁盘以文件形式存储，且无论在映射逻辑执行计划阶段，还是优化逻辑执行计划阶段，树的修改都是以替换已有节点的方式进行的。

```
Object <= TreeNode 	<= Expression
		 			<= QueryPlan <= LogicalPlan
		 			  			 <= SparkPlan
```

> TreeNode 有两个子类继承体系，即 QueryPlan 和 Expression 体系。QueryPlan 类下又包含逻辑算子树（LogicalPlan）和物理执行算子树（SparkPlan）两个重要子类。

TreeNode 的基本操作：

- `collectLeaves`：获取当前 TreeNode所有叶子结点
- `collectFirst`：先序遍历所有节点并返回第一个满足条件的节点
- `withNewChildren`：将当前节点的字节点替换为新的子节点
- `transformDown`：用先序遍历方式将规则作用于所有节点
- `transformUp`：用后续遍历方式将规则用于所有节点

**class Origin** 提供了 `line` 和 `start` 两个构造参数，分别代表行号和偏移量，可以用于定位到 sql 中的行数和起始位置，便于调试。

### 2.2.3 Expression 体系

> **Expression 指的是不需要触发执行引擎而能够直接进行计算的单元。**

> 在 Expression 类中主要定义了5个方面的操作，包括基本属性、核心操作、输入输出、字符串表达和等价性判断。
>

- Expression 的基本属性：
  - **foldable**：用来标记表达式是否能在查询执行之前直接静态计算。
  - **deterministic**：该属性用来标记表达式是否为确定性的，即每次执行 `eval`函数的输出是否都相同。
  - **nullable**：用来标记表达式是否可能输出Null值。
  - **semanticEquals**：判断两个表达式在语义上是否等价。

- 输入输出：
  - flatArgument	
  - **references**：返回值为 `AttributeSet` 类型，表示该 Expression 中会涉及的属性值，默认情况为所有子节点中属性值的集合。
  - dataType
  - checkInputDataType

- 字符串表示

- 核心操作
  - **eval**: 实现了 Expression 对应的处理逻辑，也是其他模块调用该 Expression 的主要接口。
  -  
  - doGenCode

- 等价性判断
  - **canonicalized**：返回经过规范化处理后的表达式。
  - semanticHash
  - semanticEquals

> 常用的 Expression

- **Nondeterministic** 接口：具有不确定性的 Expression， 典型实现类有： Rand。
- **Unevaluable** 接口：非可执行的表达式，调用其 `eval` 函数会抛异常，主要用于生命周期不超过逻辑计划解析和优化阶段的表达式，例如 `Star(*)` 表达式在解析阶段就会被展开成具体的列集合。
- **CodegenFallback** 接口：不支持代码生成的表达式，某些表达式涉及第三方实现（例如Hive的UDF）等情况，无法生成 Java 代码，此时通过 CodegenFallback 直接调用，该接口中实现了具体的调用方法。
- **LeafExpression**：叶子节点类型的表达式，不包含任何子节点；
- **UnaryExpression**：一元类型表达式，只含有一个子节点，例如 Abs 操作、UpCast 表达式；
- **BinaryExpression**：二元类型表达式，包含两个子节点，例如加减乘除操作；
- **TernaryExpression**：三元类型表达式，包含三个子节点，例如一些字符串操作函数。

## 3. 内部数据类型体系

```
AbstractDataType <= AnyDataType
				 <= TypeCollection
				 <= DataType <= StructType
                 		   	 <= MapType
                 			 <= ArrayType
                 			 <= NullType
                 			 <= ObjectType
                 			 <= CalenderIntervalType
                 			 <= UserDefinedType <= PythonUserDefinedType
                 			 <= AtomicType <= StringType
                 			 			   <= DataType
                 						   <= BinaryType
                 						   <= BooleanType
                 						   <= TimestampType
                 						   <= NumericType <= FractionType <= FloatType
                 						   								  <= DoubleType
                 						   								  <= DecimalType
                 						                  <= IntegeralType <= ByteType
                 						                  			       <= IntegerType
                 						                  			       <= LongType
                 						                  			       <= ShortType
```

# 3. Spark SQL 编译器Parser

## 3.1 DSL 与 ANTLR

> DSL (Domain Specific Language) 领域特定语言，如SQL。
>
> ANTLR (Another Tool for Language Recognition) 是目前活跃的语法生成工具。

一个系统中 DSL 模块的实现需要涉及两方面工作：

- 设计语法和语义，定义DSL中具体的元素；
- 实现词法分析器（Lexer）和 语法分析器（Parser），完成对DSL的解析，最终转换为底层逻辑来执行。

Spark SQL 使用的语法生成工具是 ANTLR4，其功能为：

- 自动根据 **g4** 文件中定义的 DSL 语法和语义，构建语法分析树

- 自动生成基于监听器（Listener）和访问者（Visitor）模式的树遍历器


SparkSqlParser 主要采用访问者模式，树遍历器会对语法树中的每个节点（Context对象）调用对应的访问者（Visitor）对象，生成 LogicalPlan （Unresolved LogicalPlan）并返回。

## 3.2 SparkSqlParser 与 AstBuilder

```
ParseInterface <= AbstractSqlParser <= CatalystSqlParser (+AstBuilder)
									<= SparkSqlParser (+SparkSqlAstBuilder)
SqlBaseBaseVisitor <= AstBuilder <= SparkSqlAstBuilder
```

## 3.3 抽象语法树

**在Catalyst中，SQL 语句经过解析，生成的抽象语法树节点都以 Context 结尾命名**。

## 3.4 自定义语法

### 3.4.1 添加语法

- 在 SqlBase.4g 文件中添加语法

```
./sql/catalyst/src/main/antlr4/org/apache/spark/sql/catalyst/parser/SqlBase.g4
```

- 需要在三个地方添加语法

1. 在 **nonReserved** 部分添加 新的关键字

```
FLY
```

2. 在 **nonReserved** 部分下面添加 关键字映射

```
FLY:"FLY";
```

3. 然后在 **Statement** 部分添加语法， 之后会根据 # 后面的名称 showFly 生成对应的 visitShowFly 方法

```
SHOW FLY ON TABLE sourceTable=tableIdentifier                				#showFly
```

### 3.4.2 编译 catalyst 模块

- 执行以下命令编译指定模块

```
./build/mvn -DskipTests clean package -pl :spark-catalyst_2.11 -am
```

- 编译之后，会自动生成以下源码文件

```
./sql/catalyst/target/generated-sources/antlr4/org/apache/spark/sql/catalyst/parser/
SqlBaseBaseListener.java	SqlBaseLexer.java		SqlBaseParser.java
SqlBaseBaseVisitor.java		SqlBaseListener.java	SqlBaseVisitor.java
```

- 在 `SqlBaseBaseVisitor.java` 文件中会生成对应  showFly 的 `visitShowFly()` 抽象方法
- 在 `SparkSqlParser.scala`  的 `class SparkSqlAstBuilder` 中实现 `visitShowFly()`  函数，因为这个类的父类是 `SqlBaseBaseVisitor.java`

```
./sql/core/src/main/scala/org/apache/spark/sql/execution/SparkSqlParser.scala
```

### 3.4.3 转换生成的语法树

- 生成的 `visitShowFly()` 的入口参数为 语法树中的Context 对象，返回值类型为 LogicalPlan 类型，在这个函数中完成这两个类型的转换即可。具体转换过程可参照spark源码中存在的其他转换过程。

- 最后重新编译 spark-sql 模块

```
./build/mvn -DskipTests clean package -pl :spark-sql_2.11 -am
```

# 4. LogicalPlan

> Spark SQL Logical Plan 阶段主要分为三个步骤：
>
> - 由 **SparkSqlParser** 中的 **AstBuilder** 执行节点访问（从根节点开始递归调用），将语法树的各种Context节点转换成对应的 LogicalPlan 节点（访问者模式，将 Parser 解析的 Context 对象传入对应的 visit 方法中，完成 LogicalPlan 的生成并返回），从而成为一棵未解析的逻辑算子树（**Unresolved LogicalPlan**），此时 LogicalPlan 不包含数据信息与类信息。
> - 由**Analyzed** 将一系列的规划作用在 Unresolved LogicalPlan 上，对树上的节点绑定各种数据信息，生成解析后的逻辑算子树，**Analyzed LogicalPlan** 。
> - 由**Optimiter** 将一系列优化规则作用到上一步生成的 LogicalPlan 树中，在确保结果正确的前提下改写其中的低效结构，生成优化之后的逻辑算子树，**Optimized LogicalPlan**。

## 4.1 QueryPlan 基本信息

```
Object <= TreeNode <= QueryPlan <= LogicalPlan
```

- 在 QueryPlan 的各个节点中， 包含了各种 Expression 对象 ，各种逻辑操作一般由 Expression 对象完成，Expression不需要驱动直接执行，而QueryPlan需要驱动执行。

## 4.2 LogicalPlan 基本操作和分类

基本操作包含：对数据表、表达式、schema和列属性等类型的解析。

根据子节点数目，绝大部分LogicalPlan可以分为三类：LeafNode类型（不存在子节点），UnaryNode 类型（一元节点），BinaryNode类型（包含两个子节点）。

### 4.2.1 LeafNode 类型的 LogicalPlan

LeafNode 类型的 LogicalPlan 节点对应数据表和命令（Command）相关的逻辑。

RunnableCommand 是直接运行的命令，主要涉及12种情形，包括 Database 、Table 、View、DDL、Function、Resource相关命令。

### 4.2.2 UnaryNode 类型的 LogicalPlan

主要对数据的逻辑转换，包括过滤等。

主要分为4个类别：

- 用来定义重分区（repartitioning）操作的 3 个 UnaryNode，即 RedistributeData 及其两个子类 SortPartition 和 RepartitionByExpression，主要针对现有分区和排序的特点不满足的场景。

- 脚本相关的转换操作（ScriptTransformation），用特定的脚本对输入数据进行转换。
- Object 相关的操作（ObjectConsumer）
- 基本操作算子 （basicLogicalOperators），涉及 Project、Filter、Sort 等各种常见的关系算子。

### 4.2.3 BinaryNode 类型的 LogicalPlan 

主要对数据的组合关联操作，包括 Join 算子等。

BinaryNode类型节点种比较复杂且重要的是 **Join** 算子

### 4.2.4 其他类型的 LogicalPlan

主要有三个直接继承自 LogicalPlan 的逻辑算子节点：ObjectProducer、Union、EventTimeWatermark 逻辑算子。

**Union** 算子的使用场景比较多。

## 4.3 AstBuilder 机制： Unresolved LogicalPlan 生成





# 5. Physical Plan

