# 1. MapReduce

## 1.1 Anatomy of a MapReduce Job Run

> 可以调用 `submit()` 或者 `waitForCompletion()` 提交一个MapReduce任务。`waitForCompletion()` 内部调用了 `submit()` 函数。

> 总共有5个独立部分: client、ResourceManager、NodeManager、ApplicationMaster、HDFS

### 1.1.1 Job Submission

> `submit()` 函数会创建一个内部实例 `JobSubmitter`，并且会调用这个类的 `submitJobInternal()` 函数。
> `JobSubmitter` 实例会完成以下事情：
>
> 1. 向 ResourceManager 请求一个新的 application ID;
> 2. 检查 job 的输出有效性；
> 3. 计算 job 的输入切片；
> 4. 拷贝 job 所需要的资源文件到HDFS中以 application ID命名的目录中，包括 JAR 文件、配置文件、输入切片；
> 5. 通过调用 `submitApplication()` 提交 job。 
### 1.1.2 Job Initialization

> 当 client 调用 `submitApplication()` 方法时，ResourceManager 分发请求给 YARN 调度器，YARN调度器会分配一个 container，并在 NodeManager 的管理下在这个container中运行一个 `application master` 进程。

> `application master` 是一个名叫 `MRAppMaster` 的Java进程，这个进程会创建多个对象来检测任务进度；然后，会从共享文件系统中获取输入分片；之后会为每个输入分片创建一个 map 任务，以及根据 `setNumReduceTasks(n)` 函数设置的值创建n个reduce 任务。

> `application master` 会决定如何运行 MapReduce任务。如果任务很小，`application master` 会直接在本地运行map和reduce任务，否则会向ResourceManager请求新的计算资源。

### 1.1.3 Job Assignment

> 直到 5% 的map任务完成之后，才会开始请求 reduce 任务。

> reduce 任务可以在集群的任何地方运行，但是map任务需要受到数据本地性限制。

> 每个 map 任务和 reduce 任务默认会分配 1024 MB内存和一个虚拟核心。

### 1.1.4 Task Execution

> 执行 map 和 reduce 任务的进程叫 `YarnChild` ，是运行在一个专用 JVM中，这样会将map和reduce程序的bug进行隔离，不会影响整个NodeManager崩溃。在运行任务前，`YarnChild` 会首先获得需要的资源文件。

### 1.1.5 Progress and Status Updates

> MapReduce 任务通常是个长时间任务，有必要向客户端实时反馈任务进度。对于 map 任务，就是已经被处理的输入所占的比例；对于reduce任务，虽然有些复杂，但是也可以根据已经被处理了的输入的比例来进行评估。

### 1.1.5 Job Completion


## 1.2 Failure

- `YarnChild` 中运行 map 和 reduce 任务，并向 `AppMaster` 报告进度；
- `AppMaster` 检测整个MapReduce任务，并由 `NodeManager` 管理；
- `NodeManager` 负责运行并监控 `AppManager`，同时向 `ResourceManager` 发送心跳。

> 错误主要有四个部分

### 1.2.1 Task Failure

> 大部分常见错误都是在map和reduce任务中用户代码抛出的运行时错误。一旦发生这种错误，在exit之前，`YarnChild`会将错误报告给`Application Master`， 错误会最终被写入用户日志中，`Application Master`会标记任务 *failed* 并且会释放container的所有资源。

> `Application Master` 会给失败的任务重新分配计算资源，当失败四次之后，这个任务就不会再进行尝试了，则整个job会失败。


### 1.2.2 Application Master Failure

> `Application Master` 失败后会尝试重连2次，超过这个次数，`ResourceManager` 会重新运行一个`ApplicationMaster`，（这个过程是要在 `NodeManager` 管理下进行的）并根据Job日志恢复任务状态。


### 1.2.3 Node Manager Failure

> `ResourceManger` 如果10分钟没有收到一个`NodeManager`的心跳，则认为这个`NodeManager`不可用，将会被移除`ResourceManager`的集群。

> 如果一个NodeManager的失败次数过多，将会被列入黑名单。

### 1.2.4 Resource Manager Failure

> `ResourceManager` 一旦发生错误是很严重的，导致整个YARN都无法运行，则需要配置YARN的HA，也就是配置两个`ResourceManager`。


## 1.3 Shuffle and Sort

> `Shuffle` 是 MapReduce的核心

### 1.3.1 Map Side

> 每个map任务对应一个 `input split`, 并且对应一个环形缓冲区，用来保存map任务的输出。 环形缓冲区的默认大小是100MB，可以通过 `mapreduce.task.io.sort.mb` 属性来设置这个大小。当环形缓冲区写入的内容大小超过指定阈值之后，默认是80%，也就是80MB，一个后来进程将会将缓冲区的内容溢写到磁盘中，与此同时，map任务的输出还会不断被写入到环形缓冲区中。

> - **分区**：在溢写到磁盘之前，进程首先会根据最终被发送到的 `reducers` 将数据划分到不同的区；
> - **排序**：在每个分区中，后台进程会按照key对数据进行排序；
> - **合并**：如果有combiner函数，会在排序后进运行这个函数。

> 当每次缓冲区到达溢写阈值时，一个新的溢写文件将会被创建，也就是当整个map任务结束之后，会产生多个分区、排序好了的溢写文件，程序会将这多个文件合并成一个分区并且排序后的输出文件。

> 如果溢写文件数目不少于3个，则会**再次**调用combiner函数，否则不会再调用这个函数，也可以通过设置 `mapreduce.map.combine.minspills` 参数来控制这个阈值。

> 可以将 `mapreduce.map.output.compress` 参数设置为 *true*，map输出写入磁盘的过程中会进行压缩，通过设置 `mapreduce.map.output.compress.codec` 来指定不同的压缩器。


### 1.3.2 Reduce Side


> map任务完成任务后，会通过心跳机制向AppMaster报告，AppMaster会保存每个输出文件以及所在节点的映射关系，reduce会定时向AppMaster获取map的输出，直到全部被取完。

- **copy phase** : reduce任务会在每个map任务结束后拷贝map任务的输出，reduce会默认启动5个拷贝线程专门用于并行化拷贝任务。拷贝线程数目可以通过设置 `mapreduce.reduce.shuffle.parallelcopies`。Map的输出首先会被拷贝到reduce任务JVM的内存中，当超过指定阈值时，会被溢写到磁盘中。如果有combiner，则会在溢写之前调用combiner函数。
- **merge phase** : 随着拷贝在磁盘上的积累，一个后台进程会启动用来将这些溢写文件合并成更大的文件，这样可以在最终合并的时候节省时间。在这个过程中，压缩文件会被解压。当所有map的输出文件被拷贝完后，reduce任务会合并所有文件，并且保留其排序顺序。合并过程按照round进行，50个文件，默认每10个作为一个round，先进行5次round，生成5个临时文件，然后最终再进行一次round将这5个临时文件进行合并。
- **reduce phase** : 会对合并之后的文件调用reduce函数，并将输出保存到HDFS上。


### 1.3.3 Configuration Tuning



## 1.4 Task Execution


> `OutputCommitter`

---


# 2. MapReduce Types and Formats

## 2.1 MapReduce Types

```
map: (K1, V1) -> list(K2, V2)
combiner: (K2, list(V2)) -> list(K2, V2)
partition: (K2, V2) -> integer
reduce: (K2, list(V2)) -> list(K3, V3)
```

> mapper的输出类型和reducer的输入类型必须一致

> 分区数等于reducer的数目，默认的分区函数是HashPartitioner

> reducer数默认为1， mapper的数目默认等于input split的数目

> 设置reducer数目的原则是：每个reducer运行5分钟，并且处理至少一个HDFS block的output。

## 2.2 Input Formats

- Mapper 中的 Context 对象。

### 2.2.1 Input Splits and Records

- **注意** ： 
    - `InputSplit` 对象只是包含了数据的引用，但不包含真正的数据。
    - 编写MapReduce应用的时候，不要求编程者直接处理`InputSplit`对象，`InputFormat`对象负责创建 `input splits`并划分`records`。

```
public abstract class InputFormat<K, V> {
    public abstract List<InputSplit> getSplits(JobContext context) throws IOException, InterruptedException;
    public abstract RecordReader<K, V> createRecordReader(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException;
}
```

#### 2.2.1.1 FileInputFormat

> FileInputFormat类是所有以文件为输入的InputFormat基类。它包含两个部分：**指定了作为任务输入所包含的所有文件**、**实现了从输入文件中生成splits的方法**。

#### 2.2.1.2 FileInputFormat input paths

> 通过设置 Paths 和 Filters 指定job的输入。

```
public static void addInputPath(Job job, Path path)
public static void addInputPaths(Job job, String commaSeparatedPaths)
public static void setInputPaths(Job job, Path... inputPaths)
public static void setInputPaths(Job job, String commaSeparatedPaths)
```
> 指定目录的话，这个目录下应该都是文件，如果包含子目录，也会被当做文件处理，则会报错。

> 可以通过 **setInputPathFilter()** 函数设置一个文件过滤器，但无论调用与否，都会执行默认的过滤器，会过滤掉隐藏文件。

#### 2.2.1.3 FileInputFormat input splits

> 一个split的大小一般等于HDFS block的大小，如果输入文件小于这个大小，则不会被split。

> 可以设定 **minimumSize** 和  **maximumSize** ，但split最终的大小由这个公式决定。

```
max(minimumSize, min(maximumSize, blockSize))
```

#### 2.2.1.4 Small files and CombineFileInputFormat

> Hadoop在多个小文件上的处理效果没有在少数大文件上的处理效果好。对于多个小文件的情况时，可以使用 `CombineFileInputFormat` 类。

> 为了避免处理多个小文件的情况， 可以使用 `sequence file`  ，key就是文件名，value就是文件内容。但是，如果多个文件已经在HDFS上了，则最好使用 `CombineFileinputForamt` 。

#### 2.2.1.5 Preventing splitting

> 有时候不希望对文件进行split，这时候可以通过两种方式：① 增加 **minimum size** ； ② 覆盖 **isSplitable()** 方法并返回false。

```
public class NonSplittableTextInputFormat extends TextInputFormat {
    @Override
    protected boolean isSplitable(JobContext context, Path file) {
        return false;
    }
}
```

#### 2.2.1.6 File information in the mapper

> 可以在Mapper中调用Context对象的 **getInputSplit()** 方法获得 input split的存储路径等信息。

#### 2.2.1.7 Processing a whole file as a record

> 将整个文件作为一个record进行处理，需要两个步骤：① 覆写  **isSplitable()** 方法并返回 **false** ; ② 覆写  **createRecordReader()** 方法并返回自定义的 **RecordReader** 对象。


### 2.2.2 Text Input

#### 2.2.2.1 TextInputFormat

> **TextInputFormat** 是默认的 **InputFormat** 。 key 为偏移量，value为每一行内容。

>  实际上，Split可能是跨block的，但是这种情况一般不显著。

#### 2.2.2.2 Controlling the maximum line length

> 可以通过设置 `mapreduce.input.linerecordreader.line.maxlength` 来控制每一行最大的长度，以防止内存溢出等错误。

#### 2.2.2.3 KeyValueTextInputFormat

> 可以通过 `mapreduce.input.keyvaluelinerecordreader.key.value.separator` 指定 **KeyValueTextInputFormat**  的分隔符，默认是 tab 键。

#### 2.2.2.4 NLineInputFormat

> 通过设置 `mapreduce.input.lineinputformat.linespermap` 来指定 **NLineInputFormat** 的 **N**， 即每个mapper读取N行，也就是每个 input split 包含N行数据，默认是N=1

#### 2.2.2.5 XML

> 通过  **StreamXmlRecordReader** 来读取XML格式的文件


### 2.2.3 Binary Input

> Hadoop MapReudce 不仅能处理文本数据，还能处理二进制数据。

#### 2.2.3.1  SequenceFileInputFormat

> 序列文件存储任意实现了序列化框架的类型。

#### 2.2.3.2 SequenceFileAsTextInputFormat

> SequenceFileAsTextInputFormat 是 SequenceFileInputFormat 变种，只是将key 和 value转换为Text类型。 

#### 2.2.3.3 SequenceFileAsBinaryInputFormat

> SequenceFileAsBinaryInputFormat 是 SequenceFileInputFormat 的变种，只是将key和value改为二进制对象。

#### 2.2.3.4 FixedLengthInputFormat

> 如果record没有分隔符，可以指定固定长度的record。



### 2.2.4 Multiple Inputs

> MultipleInputs 类可以给不同的输入路径指定不同的InputFormat 和Mapper。下面是个例子：

```
MultipleInputs.addInputPath(job, ncdcInputPath, TextInputFormat.class, MaxTemperatureMapper.class);
MultipleInputs.addInputPath(job, metOfficeInputPath, TextInputFormat.class, MetOfficeMaxTemperatureMapper.class);
```

### 2.2.5 Database Input (and Output)


> DBInputFormat 基于JDBC， 可以从关系型数据库中读取数据。

> 因为关系型数据库不能分块，所以不能启动多个mapper。这种方式最好加载相对较小的数据集。对于数据的迁移，最好使用Sqoop。

```
Mapper { 
    run(){
        map(context.getKey(), context.getValue(), context)
    }
}
Context {
    getKey(){
        return recordReader.getKey()
    }
    getValue(){
        return recordReader.getValue()
    }
}
RecordReader {
    //对 InputSplit 对象进行处理   
    getKey(){
    }
    getValue(){   
    }
}
```

## 2.3 Output Formats

- Reducer 中的 Context 对象

### 2.3.1 Text Output

> 默认的 OutputFormat

### 2.3.2 Binary Output

### 2.3.3 Multiple Outputs

> 默认情况下，一个partition对应一个reducer，一个reducer对应一个输出文件，但是可以通过MultipleOutputs让一个reducer将输出写入多个文件。

### 2.3.4 Lazy Output

### 8.3.5 Database Output

# 3. Other

## 3.1 Map

- **输入分块大小**：Map的输入分块大小默认和HDFS的block大小相同，即128M；
- **Map任务进程** : Map任务进程和对应的输入分块应该在同一台节点上，这叫数据局部性优化（data locality optimization）；如果这个输入分块的所有副本所在的节点都已经在跑其他的Map任务的时候，则会寻找同一个机架上的其他节点运行这个Map任务；如果同一个机架上也没有闲置的节点，则会寻找其他机架上的节点。
- **输出文件**：Map的输出文件只会保存在本地磁盘，而不会上传到HDFS中。

## 3.2 Reduce

- **输入** ： Reduce 任务是没有数据本地化优势的，其输入是所有map的输出，并通过网络传输获取的。
- **reduce任务**：三种情况：① 可以有单个reduce，对应一个输出文件；② 多个reduce，map中的partition函数会产生不同的分区，每个分区对应一个reduce，每个reduce对应一个输出文件；③ 0个reduce.
- **输出**： 对于Reduce输出的每个HDFS block，第一个副本会存在本地节点，其他副本会保存在其他机架上的节点以保证可靠性。

## 3.3 Combiner函数

- 是为了减少数据网络传输，提升MapReduce性能的一种优化方法，在map端运行，但不能改变map的输出类型。

- **注**：有时候可以将Combiner完全看成map端的reduce,内部实现与reduce完全一样，但有时候是与reduce的实现不一样的。

# 4  MapReduce 实例

## 4.1 Word Count

```java
public class WCMapper extends Mapper<LongWritable, Text, Text, LongWritable>{
	@Override
	protected void map(LongWritable key, Text value,
			Mapper<LongWritable, Text, Text, LongWritable>.Context context)
			throws IOException, InterruptedException {
		//====================================================
		String v1 = value.toString();
		String[] parts = v1.split(" ");
		for(String part : parts){
			context.write(new Text(part), new LongWritable(1));
		}
		//===================================================
	}
}
public class WCReducer extends Reducer<Text, LongWritable, Text, LongWritable>{
	@Override
	protected void reduce(Text key, Iterable<LongWritable> values,
			Reducer<Text, LongWritable, Text, LongWritable>.Context context)
			throws IOException, InterruptedException {
		//=============================================
		long count = 0;
		for(LongWritable value : values){
			count += value.get();
		}
		context.write(key, new LongWritable(count));
		//=============================================
	}	
}
public class WordCount {
	public static void main(String[] args) {
		try {
			Job job = Job.getInstance(new Configuration());
			job.setJarByClass(WordCount.class);
			// set mapper
			job.setMapperClass(WCMapper.class);   //set mapper
			job.setMapOutputKeyClass(Text.class); //set output key type
			job.setMapOutputValueClass(LongWritable.class);  //set output value type
			FileInputFormat.setInputPaths(job, new Path(args[0])); //set input data path
             // set reducer
			job.setReducerClass(WCReducer.class);  //set reducer
			job.setOutputKeyClass(Text.class);		//set output key type
			job.setOutputValueClass(LongWritable.class); //set output value type
			FileOutputFormat.setOutputPath(job, new Path(args[1])); //set output data path
			//set combiner
			job.setCombinerClass(WCReducer.class); //设置combiner（在map端执行的特殊reducer）
			job.waitForCompletion(true);  //submit job 		
		} catch (Exception e) {
			e.printStackTrace();
		}
	}	
}
```

## 4.2 Data Count

> 统计流量数据

```java
public class DataBean implements WritableComparable<DataBean>{
	private String telephone;  //手机号
	private long upStream;    //上行流量
	private long downStream;  //下行流量
	private long sumStream;   //总流量
	public DataBean(){
	}
	public DataBean(String telephone, long upStream, long downStream) {
		super();
		this.telephone = telephone;
		this.upStream = upStream;
		this.downStream = downStream;
		this.sumStream = upStream + downStream;
	}
	@Override
	public String toString() {
		return upStream + "\t" + downStream + "\t" + sumStream;
	}
	//// getter and setter
    //// ............
	@Override
	public void write(DataOutput out) throws IOException {
		out.writeUTF(telephone);
		out.writeLong(upStream);
		out.writeLong(downStream);
		out.writeLong(sumStream);
	}
	@Override
	public void readFields(DataInput in) throws IOException {
		this.telephone = in.readUTF();
		this.upStream = in.readLong();
		this.downStream = in.readLong();
		this.sumStream = in.readLong();
	}
	/**
	 * 自定义排序规则
	 * 先按upstream排序，再按downstream排序
	 */
	@Override
	public int compareTo(DataBean arg0) {
		if(this.upStream == arg0.upStream){
			return this.downStream > arg0.downStream ? -1 : 1;
		} else {
			return this.upStream > arg0.upStream ? 1 : -1;
		}
	}
}
```

```java
public class DCMapper extends Mapper<LongWritable, Text, Text, DataBean>{
	@Override
	protected void map(LongWritable key, Text value,
			Mapper<LongWritable, Text, Text, DataBean>.Context context)
			throws IOException, InterruptedException {		
		String[] parts = value.toString().split("_");
		String telephone = parts[1].trim();
		long upStream = Long.parseLong(parts[7].trim());
		long downStream = Long.parseLong(parts[8].trim());
		context.write(new Text(telephone), new DataBean(telephone, upStream, downStream));
	}
}
public class DCReduce extends Reducer<Text, DataBean, Text, DataBean>{
	@Override
	protected void reduce(Text key, Iterable<DataBean> values,
			Reducer<Text, DataBean, Text, DataBean>.Context context)
			throws IOException, InterruptedException {
		long upSumStream = 0;
		long downSumStream = 0;
		for(DataBean dataBean : values){
			upSumStream += dataBean.getUpStream();
			downSumStream += dataBean.getDownStream();
		}
		context.write(key, new DataBean(key.toString(), upSumStream, downSumStream));
	}
}
// 自定义分区类，默认分区类为 HashPartition
public class ProvidePartition extends Partitioner<Text, DataBean> {
		private static Map<String,Integer> providerMap = new HashMap<String, Integer>();
		static {
			providerMap.put("135", 1);
			providerMap.put("136", 1);
			providerMap.put("137", 1);
			providerMap.put("138", 1);
			providerMap.put("139", 1);
			providerMap.put("150", 2);
			providerMap.put("159", 2);
			providerMap.put("182", 3);
			providerMap.put("183", 3);
		}
		@Override
		public int getPartition(Text key, DataBean value, int numPartitions) {
			String account = key.toString();
			String sub_acc = account.substring(0, 3);
			Integer code = providerMap.get(sub_acc);
			if(code == null){
				code = 0;
			}
			return code;
		}
}
public class DataCount {
	public static void main(String[] args) {
		try {
			Job job = Job.getInstance(new Configuration());
			job.setJarByClass(DataCount.class);
			// map
			job.setMapperClass(DCMapper.class);
			FileInputFormat.setInputPaths(job, new Path(args[0]));
			// reduce
			job.setReducerClass(DCReduce.class);
			job.setOutputKeyClass(Text.class);
			job.setOutputValueClass(DataBean.class);
			FileOutputFormat.setOutputPath(job, new Path(args[1]));
			// partition
			job.setPartitionerClass(ProvidePartition.class);  //set partition
			//默认只启动一个 reducer
//			job.setNumReduceTasks(Integer.parseInt(args[2]));  // >= partition numbers
			//一个reducer写入一个文件
			job.waitForCompletion(true);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

- 如果想要排序，使用 MapReduce 内建的排序机制.。将 上一个 MapReducer的输出作为这个MapReducer的输入。

```java
public class SortStep {
	/*将databean作为key2，则通过hadoop的排序方法进行排序*/
	private static class SortMapper extends Mapper<LongWritable, Text, DataBean, NullWritable>{
		protected void map(LongWritable key, Text value, Mapper<LongWritable,Text,DataBean,NullWritable>.Context context) 
				throws IOException ,InterruptedException {
			String line = value.toString();
			String[] fields = line.split("\t");
			String tel= fields[0];
			long upStream = Long.parseLong(fields[1]);
			long downStream = Long.parseLong(fields[2]);
			context.write(new DataBean(tel, upStream, downStream), NullWritable.get());
		};
	}
	private static class SortReducer extends Reducer<DataBean, NullWritable, Text, DataBean>{
		@Override
		protected void reduce(DataBean bean, Iterable<NullWritable> values,
				Reducer<DataBean, NullWritable, Text, DataBean>.Context context)
				throws IOException, InterruptedException {
			context.write(new Text(bean.getTelephone()), bean);
		}
	}
	public static void main(String[] args) {
		try {
			Configuration config = new Configuration();
			Job job = Job.getInstance(config);
			job.setJarByClass(SortStep.class);
			// map
             job.setMapperClass(SortMapper.class);
			job.setMapOutputKeyClass(DataBean.class);
			job.setMapOutputValueClass(NullWritable.class);
			FileInputFormat.setInputPaths(job, new Path(args[0]));			
			// reduce		
			job.setReducerClass(SortReducer.class);
			job.setOutputKeyClass(Text.class);
			job.setOutputValueClass(DataBean.class);
			FileOutputFormat.setOutputPath(job, new Path(args[1]));
			job.waitForCompletion(true);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

