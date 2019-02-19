# 1. Spark入门

## 1.1 环境配置与部署

- [环境配置详情](../环境配置.md)
- [Spark 官网](https://spark.apache.org/docs/2.0.0/) 右上角 **Deploying** 有四种部署方式：本地模式、[Spark Standalone](https://spark.apache.org/docs/2.0.0/spark-standalone.html)、[YARN](https://spark.apache.org/docs/2.0.0/running-on-yarn.html)、[Mesos](https://spark.apache.org/docs/2.0.0/running-on-mesos.html)

### 1.1.1 本地模式部署

- 直接运行， 进入命令行

```shell
$ ./bin/spark-shell
```

### 1.1.2 Spark Standalone 部署

- 进入 `spark-2.0.0-bin-custom-spark/conf` 目录进行配置
- 将需要配置的文件首先修改名称(删除 .template )，然后根据 [Spark Standalone](https://spark.apache.org/docs/2.0.0/spark-standalone.html) 说明进行修改，例如：

```shell
$ mv slave.sh.template slave.sh
$ vim slave.sh
```

- `slave`
- `spark-env.sh`

```shell
JAVA_HOME=/usr/java/jdk1.8.0_201
SCALA_HOME=/usr/hadoop/scala-2.11.0
HADOOP_CONF_DIR=/usr/hadoop/hadoop-2.5.2/etc/hadoop
SPARK_MASTER_HOST=localhost
SPARK_MASTER_PORT=7077
SPARK_MASTER_WEBUI_PORT=8080
SPARK_WORKER_CORES=1
SPARK_WORKER_MEMORY=2g
SPARK_WORKER_PORT=7078
SPARK_WORKER_WEBUI_PORT=8081
SPARK_WORKER_INSTANCES=1
```

- 启动命令

```shell
$ ./sbin/start-master.sh 
$ ./sbin/start-slave.sh spark://localhost:7077
```

- 可以通过 WEB UI 访问

```shell
http://localhost:8080/
```



### 1.1.3 启动



# 2. Spark RDD

# 3. Spark Streaming

# 4. Spark 核心编程

# 5. Spark内核源码剖析

# 6. Spark 性能优化

