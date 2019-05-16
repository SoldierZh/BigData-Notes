### 从表1导入数据到表2

问题描述： 有时候需要将一个Spark SQL表的部分数据（某些分区）导入到另一张表中, 可以通过Spark SQL的`Insert into`语句：

```sql
CREATE TABLE new_table
LIKE old_table;

INSERT INTO TABLE new_table
SELECT * FROM old_table
WHERE date = '20190412' and hour='11';
```

但是如果数据量很大的时候，这个操作非常耗时。

为此可以通过 `distcp`命令：

```
hadoop distcp \
-Dmapreduce.map.memory.mb=1024 \
-update \
-skipcrccheck \  -- 跳过检验和检测
-m=2500 \    -- mapper 的数量
/old_table/date=20190412/hour=11 \ 
/new_table/date=20190412/hour=11
```

然后再通过 `msck`：

```
MSCK REPAIR TABLE new_table;
```

但这时候会存在一个问题，如果分区没有数据，则执行 `msck` 语句的时候会出错。例如分区字段为 date/hour/level， date=20190412/hour=11/level=1 下面有数据， date=20190412/hour=11/level=2 没有数据（但分区目录还是存在的），则`msck` 会出错，所以必须删除 new_table 中分区数据大小为0的分区目录：

```shell
$ hadoop dfs -du -h /new_table/date=20190412/hour=11 | sort -rn

	123        369        /new_table/date=20190412/hour=11/level=1
	0		 0		  /new_table/date=20190412/hour=11/level=2
$ hadoop dfs -rm -r /new_table/date=20190412/hour=11/app=2
```

然后再执行 `msck` 语句，则会通过：

```
MSCK REPAIR TABLE new_table;
show partitions new_table;
```

---

### Hive 表的 bucket

[参考](https://my.oschina.net/leejun2005/blog/178631) 

- 简述：对于每个 partition，Hive可以进一步划分为 bucket，bucket 是更为细粒度的数据范围划分。Hive采用对某一列的值进行哈希，然后除以 bucket 的个数求余的方式决定该条记录存放在哪个 bucket 当中。
- Bucket 优点：
  - 查询处理更加效率：如果有两张表 Ta, Tb，有一个共同的列 c，并且 Ta 和 Tb 都将 c 列作为 bucket 列， 在连接表 Ta 和 Tb 时（连接列包含了c 列），可以使用 Map 端连接 （Map-side join）高效的实现。对于JOIN操作，只需要将保存相同列值的 bucket 进行JOIN操作就可以，可以大大较少JOIN的数据量。
  - 使取样（sampling）更高效：在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。

```sql
CREATE TABLE bucketed_user (id INT) name STRING) 
CLUSTERED BY (id) INTO 4 BUCKETS; 
```

桶中的数据可以根据一个或多个列另外进行排序。由于这样对每个桶的连接变成了高效的归并排序(merge-sort), 因此可以进一步提升map端连接的效率。以下语法声明一个表使其使用排序桶： 

```sql
CREATE TABLE bucketed_users (id INT, name STRING) 
CLUSTERED BY (id) SORTED BY (id ASC) INTO 4 BUCKETS; 
```

**物理上，每个桶就是表(或分区）目录里的一个文件**。它的文件名并不重要，但是桶 n 是按照字典序排列的第 n 个文件。事实上，**桶对应于 MapReduce 的输出文件分区：一个作业产生的桶(输出文件)和reduce任务个数相同，也就是MapReduce的shuffle阶段中的Partitioner**。

- Partition 和 Bucket：
  - 一个表可以同时使用 Partition 和 Bucket
  - Partition 适用于枚举值较少的列，例如 date, hour； Bucket 适用于枚举值较多的列，例如 ID，因为 Bucket 会根据指定列计算hash值以确定 Bucket 的index，最终产生的  Bucket 的数量是一定的。

------

### SparkSQL的shuffle阶段Task数量

可以通过**spark.sql.shuffle.partitions**调节 Spark SQL 执行过程中 shuffle 的Task数量，调节的基础是spark集群的处理能力和要处理的数据量，其默认值是200。Task过多，会产生很多的任务启动开销，Task多少，每个Task的处理时间过长，容易straggle。

---

### SparkSQL的 Parquet 文件格式

[Parquet](https://spark.apache.org/docs/latest/sql-data-sources-parquet.html)

Parquet文件格式是按列存储的文件格式，并且会保存shema信息，是SparkSQL默认的文件格式(`spark.sql.sources.default`的默认值)，支持以下功能：

- 支持分区列，会在写入和读取文件时，自动识别分区列字段,`spark.sql.sources.partitionColumnTypeInference.enabled`默认值为`true`;
- Parquet的shema是可以动态变化的（动态增加列或者删除列），Parquet可以自动进行 Schema 的合并，由于这个操作有性能损耗，默认是不开启的，`spark.sql.parquet.mergeSchema` 默认值为`false`。

---

### Parquet 和 Hive 存储格式

[参考](https://spark.apache.org/docs/latest/sql-data-sources-parquet.html#hive-metastore-parquet-table-conversion)

当读取或者写入Hive数据表时，SparkSQL会尝试使用自带的 Parquet 而不是使用 Hive 的 SerDe， 以达到更好的性能。由 `spark.sql.hive.convertMetastoreParquet` 控制，默认为 `true`。

从表的 schema 处理的角度来看，Hive 和 Parquet 之间主要有两个 **区别** ：

- Hive不区分大小写，但是Parquet区分大小写；
- Hive认为每一列都可以为空（nullable）， 但是 Parquet 认为每一列都不能为空。

由于这两点区别，在将 Hive Metastore schema 表转换为 Spark SQL Parquet表时，必须将Hive Metastore schema 与 Spark SQL Parquet schema 进行协调。如下是协调规则：

- 两个 schema 中具有相同名称的字段，并且必须具有相同的数据类型，而不管是否为空。协调字段应该具有 SparkSQL Parquet 端的数据类型，以便遵循可为空性。
- 协调的 schema 恰好包含那Hive Metastore schema 中定义的那些字段：
  - 仅出现在 Parquet schema 中的任何字段都将放入已协调的 schema 中；
  - 仅出现在Hive Metastore schema 中的任何字段都将在协调 schema 中添加为可空字段。 

**Note**: Spark SQL 为了更好的性能，会缓存 Parquet 元数据。启用Hive Metastore Parquet 转换后，还会缓存这些转换表的元数据。如果这些表由Hive或其他外部工具更新，则需要手动刷新他们以确保元数据一致：

```scala
spark.catalog.refreshTable("my_table")
```

---

### Parquet OCR Kryo ?

序列化框架和文件格式的区别

---

### 序列化框架对比

序列化框架的**要点**：

1. 序列化速度：将普通对象转换为字节数组需要的时间

2. 对象压缩比：序列化后生成的字节数组所占的空间 与 原对象所占空间的比值

3. 支持的数据类型范围

4. 易用性

- 序列化速度对比

|     **工具**      |  Java   | ProtoBuf |  Kryo  |
| :---------------: | :-----: |  :------: | :----: |
|    **仅数字**     | 8733ns  |  1154ns  | 2010ns |
| **数字 + 字符串** | 12497ns |  2978ns  | 2863ns |

- 对象压缩比对比：

|     **工具**      | Java  | ProtoBuf | Kryo |
| :---------------: | :--: | :------: | :--: |
|    **仅数字**     | 392B   |   59B    | 56B  |
| **数字 + 字符串** | 494B   |   161B   | 149B |

- 支持的数据类型与易用性对比：
  - Java 序列化框架：
    - 优点：是 JDK 自带的对象序列化方式，兼容性和易用性是最好的，支持所有基本类型和任何实现了 `Serializable` 的 Class 的对象；
    - 缺点：序列化速度最低，对象压缩比最低。
  - Google ProtoBuf：定义了一套自己的数据类型，在使用 ProtoBuf时，必须单定义一个描述文件，用来完成Java对象中的基本类型和ProtoBuf自定义类型之间的一个映射。
    - 优点：数据类型基于描述文件，所以可以跨语言；在序列化时，没有记录属性的名称，而是给每个属性分配了一个id，所以性能较好。
    - 缺点：需要额外定义描述文件，并生成对应代码。
  - Kryo：Kryo的处理和Google Protobuf类似。但有一点需要说明的是，Kryo在做序列化时，也没有记录属性的名称，而是给每个属性分配了一个id，但是他却并没有GPB那样通过一个schema文件去做id和属性的一个映射描述，所以一旦我们修改了对象的属性信息，比如说新增了一个字段，那么Kryo进行反序列化时就可能发生属性值错乱甚至是反序列化失败的情况；而且由于Kryo没有序列化属性名称的描述信息，所以序列化/反序列化之前，需要先将要处理的类在Kryo中进行注册，这一操作在首次序列化时也会消耗一定的性能。另外需要提一下的就是目前kryo目前还只支持Java语言。

---

### 压缩框架对比

| 存储格式 | 优点     | 缺点                               | 是否可切分 | 建议用途 |
| -------- | -------- | ---------------------------------- | ---------- | -------- |
| GZIP     | 压缩率高 | CPU使用率高，压缩慢                | No         | 冷数据   |
| BZIP2    | 压缩率高 | CPU使用率高，压缩慢，HBase不支持   | yes        |          |
| LZO      | 压缩快   | 压缩率低，原生不支持，需要额外安装 | Yes        | 热数据   |
| LZ4      | 压缩快   | 压缩率比LZO略低                    | No         | 热数据   |
| Snappy   | 压缩快   | 压缩率低                           | No         | 热数据   |

**Note**: Snappy文件块不可拆分，但是在container file format里面的Snappy块是可以拆分的，例如Avro和SequenceFile。Snappy一般也需要和一个container file format一起使用。

---

### 文件格式对比

[参考](https://blog.csdn.net/dylanzr/article/details/84553434)

- **Avro**
  - 行存储
  - 主要目的是为了 scheme evolution （列信息的动态改变）
  - schema和数据保存在一起
- **ORC**
  - 列存储
  - 由Hadoop中RC files 发展而来，比 RC file更大的压缩比，和更快的查询速度
  - schema 存储在 footer 中
  - 不支持 schema evolution
  - 支持多事务 ACID
  - 高压缩比并包含索引
  - 为 Hive 而生，在许多 non-hive MapReduce 的大数据组件中不支持使用
- **Parquet**
  - 列存储
  - 与 ORC 类似，基于 Google Dremel
  - Schema 存储在footer中
  - 高压缩比并包含索引
  - 相比 ORC 的局限性，Parquet 支持的大数据范围更广
  - 是 Spark 中默认的存储格式，并且在Spark中可以支持 schema evolution 

**选取不同的数据格式**：

-  读写速度
- 按行读取多还是按列读取多
- 是否支持文件分割
- 压缩率
- 是否支持 schema evolution

---

