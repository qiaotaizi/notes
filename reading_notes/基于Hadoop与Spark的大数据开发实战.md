# 第一章：Hadoop初体验
大数据特征（4V）：  
1. 数据量大 Volume
2. 类型繁多 Variety：日志、音频、视频、图片
3. 价值密度低 Value：不相干信息多，要求价值提炼
4. 处理速度快 Velocity：要求实时性高。

Hadoop：一个分布式系统基础架构。主要包括：分布式文件系统HDFS、分布式计算系统Map Reduce、分布式资源管理系统YARN。

HDFS的基本原理是将数据文件以指定的块大小拆分成数据块，并将数据块以副本的方式存储到多台机器上，即使某个节点出现故障，该节点上存储的数据块副本丢失，但是在其他节点上还有对应的数据副本，所以在HDFS中即使某个节点出现问题也不会造成数据的丢失（前提是你的Hadoop集群的副本系数大于1）。

一个Map Reduce作业通常会把输入的数据集切分为若干独立的数据块，由map任务以并行的方式处理它们，对map的输出先进行排序，然后再把结果输入reduce任务，由reduce任务来完成最终的统一处理。通常Map Reduce作业的输入和输出都是使用Hadoop分布式文件系统（HDFS）进行存储，换句话说，就是Map Reduce框架处理数据的输入源和输出目的地的大部分场景都是存储在HDFS上的。

在部署Hadoop集群时，通常是将计算节点和存储节点部署在相同的节点之上。提高网络带宽利用率。“移动计算而不是移动数据”。

YARN的基本思想是将Map Reduce架构中的Job Tracker的资源管理和作业调度监控功能进行分离。可运行各种不同类型的作业，比如：Map Reduce、Spark、Tez等不同的计算框架。

# 第二章：Hadoop分布式文件系统
分布式文件系统（DFS）允许将一个文件通过网络在多台主机上以多副本（提高容错性）的方式进行存储，实际上是通过网络来访问文件，用户和程序看起来就像是访问本地的磁盘一样。

Hadoop对多种底层文件系统的整合：  
文件系统抽象类：org.apache.hadoop.fs.FileSystem  
实现：
|文件系统|URI方案|Java实现|定义|
|----|----|----|----|
|Local|file|fs.LocalFileSystem|支持客户端校验和本地文件系统。带有校验和的本地文件系统在fs.RawLocalFileSystem中实现|
|HDFS|hdfs|hdfs.DistributionFileSystem|Hadoop的分布式文件系统|
|HFTP|hftp|hdfs.HftpFileSystem|支持通过HTTP以制度方式访问HDFS，distcp经常用于在不同的HDFS集群间复制数据|
|HSFTP|hsftp|hdfs.HsftpFileSystem|支持通过HTTPS以制度方式访问HDFS|
|HAR|har|fs.HarFileSystem|构建在Hadoop文件系统之上，对文件进行归档。Hadoop归档文件主要用来减少NameNode的内存使用|
|FTP|ftp|fs.ftp.FtpFileSystem|由FTP服务器支持的文件系统|
|S3（本地）|s3a|fs.s3native.NativeS3FileSystem|基于Amazon S3的文件系统|
|S3（基于块）|s3|fs.s3.NativeS3FileSystem|基于Amazon S3的文件系统，以块格式存储解决了S3的5GB文件大小限制问题。|

用户可以使用URI方案选取合适的文件系统来实现交互。

**HDFS的优点：**
1. 适合处理超大文件。通常是指MB到TB量级的数据文件。若HDFS中存在众多小文件，则会导致集群性能下降。
2. 可运行于廉价的机器。
3. 流式访问数据。

**HDFS的缺点：**

1. 不适合低延迟数据访问。实时性、低延迟的查询使用HBase会是更好的选择。
2. 无法高效存储大量小文件。NameNode存放文件元数据，集群中的小文件过多，会导致Name Node的压力陡增，进而影响到集群的性能。可以采用Sequence File等方式对小文件进行合并，或者是使用Name Node Federation的方式来改善。

**HDFS的设计目标：**
1. 硬件错误。
2. 大规模数据集
3. 移动计算代价比移动数据代价低。

**HDFS核心概念**

1. 数据块（Block）：
HDFS默认基本存储单位。一块64MB（或128MB）。文件分割为数据块大小进行存储。若一个文件小于数据块大小，则按照实际大小占用存储空间。
2. 元数据节点（Name Node）
管理文件系统的命名空间。将所有文件和文件夹元数据保存在一个文件系统树中。  
目录结构：  
${dfs.name.dir}/current/VERSION  
保存HDFS的版本号。  
${dfs.name.dir}/current/edits  
记录写操作日志。  
${dfs.name.dir}/current/fsimage  
命名空间文件。  
${dfs.name.dir}/current/fstime  
3. 数据节点（Data Node）
真正存储数据的地方。Block存储于数据节点  
目录结构：  
${dfs.name.dir}/current/VERSION  
${dfs.name.dir}/current/blk\_\<id_1>  
${dfs.name.dir}/current/blk\_\<id_1>.meta  
${dfs.name.dir}/current/blk\_\<id_2>  
${dfs.name.dir}/current/blk\_\<id_2>.meta  
…  
${dfs.name.dir}/current/blk\_\<id_64>  
${dfs.name.dir}/current/blk\_\<id_64>.meta  
${dfs.name.dir}/current/subdir0/  
${dfs.name.dir}/current/subdir1/  
…  
${dfs.name.dir}/current/subdir63/  
其中，blk_id保存HDFS数据块，二进制数据。blk_id.meta保存数据块属性信息：版本、类型、校验和。subdir：当目录中的数据块达到一定数量，创建子文件夹保存数据块及数据块属性。
4. 从元数据节点（Secondary Name Node）
周期性地将Name Node的namespace image和edit log合并，以防日志文件过大。备份namespace image，以备name node失效恢复(checkpoint)。  
目录结构：  
${dfs.name.dir}/current/VERSION  
${dfs.name.dir}/current/edits  
${dfs.name.dir}/current/fsimage  
${dfs.name.dir}/current/fstime  
${dfs.name.dir}/previous.checkpoint/VERSION  
${dfs.name.dir}/previous.checkpoint/edits  
${dfs.name.dir}/previous.checkpoint/fsimage  
${dfs.name.dir}/previous.checkpoint/fstime  

**HDFS架构**
master/slave架构（Name Node/Data Node）。java语言开发，移植性强。
典型的部署场景：一台机器运行Name Node实例，集群中的其他机器各自运行一个Data Node实例。
副本系数：由Name Node保存的记录文件副本的数目。

**HDFS基本操作**
基本命令格式：hdfs dfs -cmd \<args>  
hadoop fs -ls 目录路径：查看指定目录下的文件。  
hadoop fs -mkdir 目录名称：创建文件夹。  
hadoop fs -put 源路径 目标存放路径：将文件上传至HDFS  
hadoop dfs -get HDFS文件路径 本地存放路径：从HDFS下载文件。  
hadoop fs -text HDFS文件路径：查看文件内容。  
hadoop fs -cat HDFS文件路径：同上  
hadoop fs -du 目录路径：查看目录下的各个文件大小，单位：字节。  
hadoop fs -rm 文件/目录路径：删除文件或空目录。  
hadoop fs -rmr 目录路径：级联删除目录及其子目录和文件。  
hadoop fs -help 命令名：查询命令说明  

**HDFS文件读取流程**
1. 开启DistributeFileSystem，访问NameNode，获取文件块所在位置。（open）
2. 访问最近位置的DataNode，获取文件块，关闭连接，到下一块最佳位置继续读取文件块，直到完成读取。（read）
3. 关闭连接资源。（close）

**HDFS写文件流程**
1. 创建文件，并执行必要的检查，在NameNode中创建一套元数据记录（create）
2. 客户端将文件数据通过FSDataOutputStream写入一个内部队列，队列中的数据按副本数写入对应的DataNode进行存储。DataNode写入后会返回确认信息，所有DataNode都确认了，队列数据会进行删除（write）
3. 关闭资源（close）

**HDFS副本摆放策略**
默认副本数为3  
第一个副本放置在上传文件的DataNode上，若集群外提交，随机选择磁盘不太慢、cpu不太忙的节点上。  
第二个副本放置在与第一副本不同机架的节点上。  
第三副本放置在与第二副本相同机架不同的不同节点上。  
若有更多副本，则随机放置。  

**副本系数**
修改副本系数不会影响已上传文件的副本数  
客户端可以主动指定副本系数  

**HDFS负载均衡**
HDFS会自动将数据从空闲空间小的DataNode上移动至空闲空间大的DataNode。当某个文件的访问量比较高时，会启动计划来为这个文件增加副本数。  

**HDFS机架感知**
不能自动感知网络拓扑，需要配置dfs.network.script参数。配置文件提供了ip到rackid的翻译。如果topology.script.file.name没有设定，则每个ip都会被翻译成/default-rack  
通过rackid计算DataNode之间的距离  
rackid表示方式：/Dx/Rx/Hx。其中，D、R表示交换机，H表示DataNode。  

**Hadoop序列化/反序列化**
通过实现Writable接口实现序列化/反序列化。  
代码套路：
```java
//定义实体类
//实现WritableComparable，如果不需要比较的话，也可以直接实现Writable
//属性可使用java数据类型，也可以使用Hadoop的数据类型，这里使用Hadoop的
public class Person implements WritableComparable<Person>{
	private Text name = new Text();
	private IntWritable age = new IntWritable();
	private Text sex = new Text();
}
```
```java
//实现write
@Override
public void write(Data Output out) throws IOException {
　　name.write(out);
　　age.write(out);
　　sex.write(out);
}
```
```java
//实现read
@Override
public void read Fields(Data Input in) throws IOException {
　　name.read Fields(in);
　　age.read Fields(in);
　　sex.read Fields(in);
}
```

```java
public class HadoopSerializationUtil {
	public static byte[] serialize(Writable writable) throws IOException {
　　　ByteArrayOutputStream out = new ByteArrayOutputStream();
　　　DataOutputStream dataout = new DataOutputStream(out);
　　　writable.write(dataout);
　　　dataout.close();
　　　return out.toByteArray();
　　}
	public static void deserialize(Writable writable, byte[] bytes) throws Exception{
　　　ByteArrayInputStream in = new ByteArrayInputStream(bytes);
　　　DataInputStream datain = new DataInputStream(in);
　　　writable.readFields(datain);
　　　datain.close();
　　}
}
```

**SequenceFile**
将一系列键值对序列化到文件。  
SequenceFile为方便查找数据，会添加额外的数据，因此100GB数据的SequanceFile实际占用大于100GB的空间。

**SequenceFile特点**
1. 支持压缩。
基于Record压缩：针对行压缩，只压缩Value部分。  
基于Block压缩：Key、Value都会压缩。  
2. 本地化任务支持：因为文件可以被切分，因此在运行Map Reduce任务时数据的本地化情况应该是非常好的；尽可能多地发起Map Task来进行并行处理，进而提高作业的执行效率。
3. 难度低：因为是Hadoop框架提供的API，所以业务逻辑一侧的修改比较简单。

**MapFile**
是排过序的SequenceFile。由两部分构成，分别是data和index。检索效率更高。  
索引记录key及其位置偏移。

# 第三章：Hadoop分布式计算框架

**Map Reduce**
是一种简化并行计算的编程模型。  
Map：分散任务。Reduce：汇总结果。  
擅长做大数据量离线处理，不擅长实时处理。  

**MapReduce不擅长的场景**
1. 实时计算：Map Reduce 无法像 My SQL 一样，在毫秒或者秒级内返回结果。
2. 流式计算：流式计算的输入数据是动态的，而 Map Reduce 的输入数据集是静态的，不能动态变化。这是因为 Map Reduce 自身的设计特点决定了数据源必须是静态的。
3. DAG（有向图）计算：多个应用程序存在依赖关系，后一个应用程序的输入为前一个应用程序的输出。在这种情况下，Map Reduce 并不是不能做，而是使用其之后，每个Map Reduce 作业的输出结果都会写入到磁盘，会造成大量的磁盘IO，降低使用性能。

map()函数以key/value对作为输入，产生另外一系列key/value对作为中间输出写入本地磁盘。Map Reduce框架会自动将这些中间数据按照key值进行聚集，且key值相同（用户可设定聚集策略，默认情况下是对key值进行哈希取模）的数据被统一交给reduce()函数处理。  
reduce()函数以key及对应的value列表作为输入，经合并key相同的value值后，产生另外一系列key/value对作为最终输出写入HDFS。

**编程模型三步曲**
1. Input：一系列k1/v1对。
2. Map和Reduce：Map：(k1,v1) -> list(k2,v2)，Reduce：(k2, list(v2))->list(k3,v3)。其中：k2/v2是中间结果对。
3. Output：一系列(k3,v3)对。

**InputFormat接口**
决定了输入文件如何被 Hadoop分块。  
getSplits(JobContext context) 方法负责将一个大数据在逻辑上分成许多片。  
createRecordReader(InputSplit split,TaskAttemptContext context)方法根据InputSplit定义的方法，返回一个能够读取分片记录的 RecordReader。

**常用的InputFormat实现类**
1. FileInputFormat：所有使用文件作为数据源的Input Format实现的基类，主要作用是指出作业的输入文件位置。
2. KeyValueTextInputFormat每一行均为一条记录，被分隔符（缺省是tab）分割为key（Text），value（Text）。

**OutputFormat接口**  
用于描述输出数据的格式，它能够将用户提供的key/value对写入特定格式的文件中。

**常用的OutputFormat实现类**
1. TextOutputFormat：每条记录写为文本行，利用它的键和值可以实现Writable的任意类型。每个key/value对由制表符进行分割。与FileOutputFormat对应的输入格式是 KeyValueTextInputFormat，它通过可配置的分隔符将key/value对文本分割。
2. SequenceFileOutputFormat：输出写为一个顺序文件。如果输出需要作为后续Map Reduce任务的输入，这将是一种好的输出格式，因为其格式紧凑，很容易被压缩。

**Combiner**
一个合并函数，为了避免map任务和reduce任务之间的无效数据传输而设置。  
使用Combiner求和、求最值，但是求平均数不可以。

**Partitioner**
Mapper任务划分数据的过程称作Partition，负责划分数据的类称作Partitioner。  
Map Reduce默认的Partitioner是HashPartitioner。

**RecordReader**
表示以怎样的方式从分片中读取一条记录，每读取一条记录都会调用一次RecordReader类。  
系统默认的RecordReader是LineRecordReader，它是TextInputFormat对应的RecordReader；而SequenceFileInputFormat对应的RecordReader是SequenceFileRecordReader。

可通过继承RecordReader自定义实现类。需要配合覆盖InputFormat的createRecordReader方法来使用自定义的RecordReader。

**MapReduce高级应用**  
***实现SQL的join语法：***
1. Map端读取所有的文件，并在输出的内容里加上标示，代表数据是从哪个文件里来的。
2. 在reduce处理函数中，按照标识对数据进行处理。
3. 然后根据key用join来求出结果直接输出。

***实现排序：***
在Map Reduce中默认可以进行排序，如果key为封装成int的Int Writable类型，那么Map Reduce按照数字大小对key排序；如果key为封装成String的Text类型，那么Map Reduce按照字典顺序对字符串排序。

***实现二次排序：***
1. Mapper任务会接收输入分片，然后不断地调用map函数，对记录进行处理。处理完毕后，转换为新的<key,value>输出。
2. 对map函数输出的<key, value>调用分区函数，将数据进行分区。不同分区的数据会被送到不同的Reducer任务中。
3. 对于不同分区的数据，会按照key进行排序，这里的key必须实现WritableComparable接口。该接口实现了Comparable接口，因此可以进行比较排序。
4. 对于排序后的<key,value>，会按照key进行分组。如果key相同，那么相同key的<key,value>就被分到一个组中。最终，每个分组会调用一次reduce函数。
5. 排序、分组后的数据会被送到Reducer节点。

***合并小文件<font color='red'> （再看看） </font>：***

1. 覆盖FileInputFormat的isSplitable方法和createRecordReader方法，isSplitable方法总是返回false，保证小文件不可分片。
2. 自定义RecordReader，进行文件读取。
3. 实现Mapper，使用相同的文件名作为key，小文件的二进制内容作为value。（不需要实现reducer）

# 第4章：Hadoop新特性

**YARN**
Apache Hadoop YARN（Yet Another Resource Negotiator，另一种资源协调者）是一种新的 Hadoop 资源管理器，是一个通用资源管理系统，可为上层应用提供统一的资源管理和调度。它的引入为集群在利用率、资源统一管理和数据共享等方面带来了很大好处。

YARN是随着Hadoop发展而催生的新框架，取代了以前Hadoop1.x中Job Tracker的角色。因为以前Job Tracker的任务过重，负责任务的调度、跟踪和失败重启等过程，而且只能运行Map Reduce作业，不支持其他编程模式，这也限制了JobTracker的使用范围，于是YARN应运而生。

YARN由Client、Resource Manager（简称RM）、Node Manager（简称NM）、Application Master（简称AM）组成，也采用Master/Slave结构，一个ResourceManager对应多个Node Manager。  
Client向Resource Manager提交任务、终止任务等。  
Application Master由对应的应用程序完成；每个应用程序对应一个Application-Master，Application Master向Resource Manager申请资源用于在NodeManager上启动相应的任务。  
Node Manager通过心跳信息向Resource Manager汇报自身的健康状况、任务执行情况、领取任务情况等。  
Map Task对应的是Map Reduce作业启动时产生的Map任务，MPI Task是MPI框架（MPI是消息传递接口，可以理解为更原生的一种分布式模型）对应的执行任务。

**YARN核心组件功能**

1. Resource Manager：整个集群只有一个，负责集群资源的统一管理和调度。处理来自客户端的请求（启动/终止应用程序）。
2. Node Manager：整个集群中有多个，负责单节点资源管理和使用。
3. Application Master：每个应用一个，负责应用程序的管理。
4. Container：对任务运行环境的抽象。

**YARN容错**

1. Resource Mananger：基于ZooKeeper实现高可用机制（High Available，HA）避免单点故障。
2. Node Manager：执行失败后，Resource Manager将失败任务告诉对应的Application Master，由Application Master决定如何处理失败的任务。
3. Application Master：执行失败后，由Resource Manager负责重启；Application Master需处理内部任务的容错问题，并保存已经运行完成的Task，重启后无需重新运行。

**HDFS Name Node 高可用机制体系架构：**
设置多个有主备关系的NameNode，主NameNode为Active NameNode，备NameNode为StandBy NameNode。通过ZKFailoverController进行健康检查和主备切换（注意，与Secondary NameNode不同，SNN用于FSImage重写）。ANN与SNN通过Journal Node通信来完成原数据信息修改的同步。

**HDFS Name Node的主备切换实现原理：**
ZKFailover Controller作为Name Node机器上一个独立的进程启动（进程名为zkfc），启动的时候会创建Health Monitor和Active Standby Elector这两个主要的内部组件。ZKFailover Controller在创建Health Monitor和Active StandbyElector的同时，也会向Health Monitor 和 Active Standby Elector 注册相应的回调方法。  
Health Monitor 主要负责检测Name Node的健康状态，如果检测到Name Node的健康状态发生变化，会回调ZKFailover Controller的相应方法进行自动的主备选举。  
Active Standby Elector主要负责完成自动的主备选举，内部封装了Zoo Keeper的处理逻辑，一旦Zoo Keeper主备选举完成，会回调ZKFailover Controller的相应方法来进行Name Node的主备状态切换。

**HDFS Name Node高可用机制环境搭建**

|部署进程\\机器节点|hadoop001|hadoop002|hadoop003|hadoop004|hadoop005|hadoop006|
|----|----|----|----|----|----|----|
|NameNode|Y|Y|||||
|DataNode|Y|Y|Y|Y|Y|Y|
|JournalNode|Y|Y|Y|Y|Y||
|zk|Y|Y|Y|Y|Y||
|zkfc|Y|Y|||||
注意：ZK和JN使用奇数节点进行部署。

**HDFS Name Node Federation**
将单节点Name Node扩展为集群，解决如下问题：
1. HDFS集群扩展性。多个Name Node分管一部分目录，使得一个集群可以扩展到更多节点，不再像Hadoop 1.0中由于内存的限制而制约文件存储数目。
2. 性能更高效。多个Name Node管理不同的数据，且同时对外提供服务，将为用户提供更高的读写吞吐率。
3. 良好的隔离性。用户可根据需要将不同业务数据交由不同Name Node管理，这样不同业务之间影响很小。

HDFS Federation中的Name Node相互独立管理自己的名字空间。这时候在DataNode上就不仅仅存储一个Block Pool下的数据，而是多个Block Pool的数据。在HDFS Federation机制下，只有元数据的管理与存放被分隔开，而真实数据的存储还是共用的。

**HDFS Snapshots：**
文件系统在某一时刻的只读镜像，可以是一个完整的文件系统，也可以是某个目录的镜像。

**HDFS REST API：**
通过开启hdfs-site.xml中的dfs.webhdfs.enabled属性节点，启用HDFS的Http访问方式。

**YARN Resource Manager自动重启：**
可以划分成两个阶段来完成：
1. Resource Manager将应用程序的状态以及其他验证信息保存到一个可插拔的状态存储中；Resource Manager重启时将从状态存储中重新加载这些信息，然后重新开始之前正在运行的应用程序，不需要用户重新提交应用程序。（Hadoop2.4.0实现）
2. 重启时通过从Resource Manager读取容器的状态和从Application Master读取容器的请求，集中重构Resource Manager的运行状态。与第一阶段不同的是，在这个阶段中，之前正在运行的应用程序将不会在Resource Manager重启后被杀死，所以应用程序不会因为Resource Manager中断而丢失工作。（Hadoop2.6.0实现）

**Resource Manager高可用机制架构**
通过Active/Standby架构模式，结合ZK实现HA。
集群搭建：
|部署进程\\机器节点|hadoop001|hadoop002|hadoop003|hadoop004|hadoop005|hadoop006|
|----|----|----|----|----|----|----|
|NameNode|Y|Y|||||
|DataNode|Y|Y|Y|Y|Y|Y|
|JournalNode|Y|Y|Y|Y|Y||
|zk|Y|Y|Y|Y|Y||
|zkfc|Y|Y|||||
|ResourceManager|Y|Y|||||
|NodeManager|Y|Y|Y|Y|Y|Y|

# 第5章：Hadoop分布式数据库
**HBase**
HBase是一个基于HDFS的面向列的分布式数据库。是一种非关系型数据库。  
HDFS基于流式数据访问，低时间延迟的数据访问并不适合在HDFS上运行。所以，如果需要实时地随机访问超大规模数据集，使用HBase是更好的选择。

HBase本质上只有插入操作，更新和删除都是使用插入方式完成的，这是由其底层HDFS的流式访问特性（一次写入、多次读取）决定的。所以在更新时总是插入一个带时间戳的新行，而删除时则插入一个带有删除标记的新行。每次的插入都有一个时间戳标记，每次都是一个新的版本，HBase会保留一定数量的版本，这个值是可以设定的。如果在查询时提供时间戳则返回距离该时间最近的版本，否则返回离现在最近的版本。

采用Master/Slaves的主从服务器结构，它由一个HMaster服务器和多个HRegion Server服务器构成。通过Zookeeper进行协调。  
HMaster负责管理所有的HRegion Server，各HRegion Server负责存储许多HRegion，每一个HRegion是对HBase逻辑表的分块。

**HRegion：**
HBase使用表（Table）存储数据集，表由行和列组成。  
当表的大小超过设定值时，HBase会自动将表划分为不同的区域（Region），每个区域称为一个HRegion，它是HBase集群上分布式存储和负载均衡的最小单位，在这一点上，表和HRegion类似于HDFS中文件与文件块的概念。  
一个HRegion中保存一个表中一段连续的数据，通过表名和主键范围（开始主键～结束主键）来区分每一个HRegion。  
每个HRegion由多个HStore组成，每个HStore对应表中一个列族（ColumnFamily）的存储。  
HStore由两部分组成：Mem Store和Store File，用户写入的数据首先放入Mem Store，当Mem Store满了以后再刷入（flush）Store File。  
Store File是HBase中的最小存储单元，底层最终由HFile实现，而HFile是键值对数据的存储格式，其实质是HDFS的二进制格式文件。  

**HRegion Server：**
负责响应用户I/O请求，向HDFS中读写数据，一台机器上只运行一个HRegion Server。  
HRegion Server包含两部分：HLog部分和HRegion部分。  
HLog用于存储数据日志，实质是HDFS的Sequence File。到达HRegion的写操作首先被追加到日志中，然后才被加入内存中的Mem Store。

**HMaster：**
每台HRegion Server都会和HMaster服务器通信。HMaster的主要任务就是告诉每个HRegion Server它要维护哪些HRegion。  
管理用户对表的增、删、改、查操作。  
管理HRegion Server的负载均衡，调整HRegion分布。  
在HRegion分裂后，负责新HRegion的分配。  
在HRegion Server停机后，负责失效HRegion Server上的HRegion迁移。

ZooKeeper存储的是HBase中的-ROOT-表和.META.表的位置，这是HBase中两张特殊的表，称为根数据表（-ROOT-）和元数据表（.META.）。.META.表记录普通用户表的HRegion标识符信息，每个HRegion的标识符为：表名＋开始主键＋唯一ID。随着用户表的HRegion的分裂，.META.表的信息也会增加，并且还可能会被分割为几个HRegion，此时可以用一个-ROOT-表来保存META的HRegion信息，而-ROOT-表是不能被分割的，也就是-ROOT-表只有一个HRegion。那么客户端（Client）在访问用户数据前需要先访问Zoo Keeper，然后访问-ROOT-表，接着访问.META.表，最后才能找到用户数据所在的位置进行访问。

**HBase数据模型：**
***表（Table）***：是一个稀疏表（不存储值为NULL的数据），表的索引是行关键字、列关键字和时间戳。  
***行关键字（Row Key）***：行的主键，唯一标识一行数据，也称行键。表中的行根据行键进行字典排序，所有对表的访问都要通过表的行键。在创建表时，行键不用也不能预先定义，而对表数据进行操作时必须指定行键，行键在添加数据时首次被确定。  
***列族（Column Family）***：行中的列被分为“列族”。同一个列族的所有成员具有相同的列族前缀。例如“course:math”和“course:art”都是列族“course”的成员。一个表的列族必须在创建表时预先定义，列族名称不能包含ASCII控制字符（ASCII码在0～31间外加127）和冒号（:）。  
***列关键字（Column Key）***：也称列键。格式为：“\<family>:\<qualifier>”，其中family是列族名，用于表示列族前缀；qualifier是列族修饰符，表示列族中的一个成员，列族成员可以在随后使用时按需加入，也就是只要列族预先存在，我们随时可以把列族成员添加到列族中去。列族修饰符可以是任意字节。  
***存储单元格（Cell）***：在HBase中，值是作为一个单元保存在系统中的，要定位一个单元，需要使用“行键+列键+时间戳”3个要素。  
***时间戳（Timestamp）***：插入单元格时的时间，默认作为单元格的版本号。

**数据结构在RDB和HBase中的区别示例（概念视图）：**
学生成绩表：  
RDB：

|pk|name|grade|math|art|
|----|----|----|----|----|
|1|jason|2|57|87|
|2|tom|1|89|80|

HBase:

<table border="1">
<tr>
<th rowspan="2">行键（name）</th>
<th rowspan="2">时间戳</th>
<th colspan="2">列族（grade）</th>
<th colspan="2">列族（cource）</th>
</tr>
<tr>
<th>列关键字</th>
<th>值</th>
<th>列关键字</th>
<th>值（单元格）</th>
</tr>
  <tr>
  <td rowspan="3">jason</td>
    <td>t6</td>
    <td></td>
    <td></td>
    <td>course:math</td>
    <td>57</td>
  </tr>
    <tr>
    <td>t5</td>
    <td></td>
    <td></td>
    <td>course:art</td>
    <td>87</td>
  </tr>
  <tr>
    <td>t4</td>
    <td>grade:</td>
    <td>2</td>
    <td></td>
    <td></td>
  </tr>
    <tr>
  <td rowspan="3">tom</td>
    <td>t3</td>
    <td></td>
    <td></td>
    <td>course:math</td>
    <td>89</td>
  </tr>
    <tr>
    <td>t2</td>
    <td></td>
    <td></td>
    <td>course:art</td>
    <td>80</td>
  </tr>
  <tr>
    <td>t1</td>
    <td>grade:</td>
    <td>1</td>
    <td></td>
    <td></td>
  </tr>
</table>
实际存储时，概念视图是按照列族来存储的，一个新的列键可以随时加入到已存在的列族中，这也是为什么列族必须在创建表时预先定义。

**HBase物理视图示例：**
<table border="1">
<tr>
<th rowspan="2">行键（name）</th>
<th rowspan="2">时间戳</th>
<th colspan="2">列族（grade）</th>
</tr>
<tr>
<th>列关键字</th>
<th>值</th>
</tr>
  <tr>
  <td>jason</td>
    <td>t4</td>
    <td>grade:</td>
    <td>2</td>
  </tr>
    <tr>
  <td>tom</td>
    <td>t1</td>
    <td>grade:</td>
    <td>1</td>
  </tr>
</table>

<table border="1">
<tr>
<th rowspan="2">行键（name）</th>
<th rowspan="2">时间戳</th>
<th colspan="2">列族（course）</th>
</tr>
<tr>
<th>列关键字</th>
<th>值</th>
</tr>
  <tr>
  <td rowspan="2">jason</td>
    <td>t6</td>
    <td>course:math</td>
    <td>57</td>
  </tr>
    <tr>
    <td>t5</td>
    <td>course:art</td>
    <td>87</td>
  </tr>
    <tr>
  <td rowspan="2">tom</td>
    <td>t3</td>
    <td>course:math</td>
    <td>89</td>
  </tr>
    <tr>
    <td>t2</td>
    <td>course:art</td>
    <td>80</td>
  </tr>
</table>
**HBase与RDB的比较：**
***数据类型：***HBase只有简单的字符串类型，它只保存字符串。而关系型数据库有着丰富的类型选择和存储方式。  
***数据操作：***HBase只有简单的插入、查询、删除、清空等操作，表和表之间是分离的，没有复杂的表和表之间的关系，所以不能也没有必要实现表和表之间的关联操作。而关系型数据库有多种连接操作。  
***存储模式：***HBase是基于列存储的，每个列族都由几个文件保存，不同列族的文件是分离的。而关系型数据库是基于表格结构和行模式存储的。
数据维护：HBase的更新操作实际上是插入了新的数据，它的旧版本依然会保留，而关系型数据库是替换修改。  
***可伸缩性：***HBase这类分布式数据库就是为了这个目的而开发出来的，所以它能够轻松地增加或减少硬件数量，并且对错误的兼容性也比较高。而关系型数据库通常需要增加中间层才能实现类似的功能。

通常是将HBase的HMaster运行在HDFS的Name Node上，而将HRegionServer运行在HDFS的Data Node上。

**HBase Shell常用命令：**
***1.create 'table_name', 'column_name1', 'column_name2', ... ,'column_nameN'***
创建表
注意：作为行键的列和时间戳列不需要在创建表时指定。

***2.list***
查看HBase中的所有表

***3.describe 'table_name'***
查看数据库列族详细描述。
列族描述示意：

|列族参数|取值|说明|
|----|----|----|
|NAME|可打印的字符串|列族名称，参考ASCII码表中可打印的字符串|
|DATA_BLOCK_ENCODING|NONE（默认）|数据块编码|
|BLOOMFILTER|NONE（默认）、ROWCOL、ROW|提高随机读的性能|
|REPLICATION_SCOPE|默认0|开启复制功能|
|VERSION|数字|列族中单元时间版本最大数量|
|COMPRESSION|NONE（默认）、LZO、SNAPPY、GZIP|压缩编码|
|MIN_VERSIONS|数字|列族中单元时间版本最小数量|
|TTL|默认FOREVER|单元时间版本超时时间，可为其指定多长时间（秒）后失效|
|KEEP_DELETED_CELLS|TRUE、FALSE（默认）|启用后避免被标记为删除的单元从HBase中删除|
|BLOCKSIZE|默认65535字节|数据块大小，数据块越小，索引越大|
|IN_MEMORY|true、false（默认）|是否列族在缓存中拥有更高的优先级|
|BLOCKCACHE|true（默认）、false|是否将数据放入读缓存|
如果需要在创建表时指定上述的某些属性，需要使用完整建表指令：
create 'table_name',{NAME=>'column_name1',VERSIONS=>versionNo},{NAME=>'column_name2',VERSIONS=>versionNo}

***4.put 'table_name' ,'row_key','column_key','value'***
向表中插入数据

***5.scan 'table_name',{COLUMNS=>[column_family1,'column_family2',...],param_name=>param_value,...}***
扫描表
若不指定大括号中的条件，则查询所有数据。

***6.get 'table_name','row_key',{COLUMNS=>['column_family1','column_family2',...],param_name=>param_value,...}***
或者 ***get 'table_name','row_key',{COLUMN=>['column_key1','column_key2',...],param_name=>param_value,...}***
根据行键查询数据。可以不指定列族或列键，表示查询全部数据

***7.delete 'table_name','row_key' ,'column_key'***
删除一个单元的数据

***8.deleteall 'table_name','row_key'***
删除一行数据

***9.alter 'table_name', param_name=>param_value,...***
增加或修改列族
其中列族名参数NAME必须提供，如果已存在则修改，否则会增加一个列族。
同时修改或增加多个列族时应以逗号分开，并且每个列族用“{}”括起来：
***alter 'table_name',{ param_name=>param_value,...},{ param_name=>param_value,...}***

***10.disable 'table_name'***
禁用表

***11.enable 'table_name'***
启用表

***12.drop 'table_name'***
删除表，只能删除禁用状态的表

从HBase 0.98版本后开始支持表名字空间，表示对表的逻辑分组。HBase默认定义两个namespace：hbase（系统内建表名字空间）和default（用户未指定表名字空间时划分到此）。  
涉及命名空间和权限管理的命令：namespace命令组、security命令组的grant、revoke、user_permission。

**HBase的Java API**
略

# 第6章：Hadoop综合实战
**集成MapReduce与HBase**
这样带来的好处是，我们既利用了MapReduce的分布式计算的优势，也利用了HDFS海量存储的特点，特别是利用了HBase对海量数据实时访问的特点。  
可以在Map Reduce作业需要输入数据时从HBase中读取，而在输出数据时，又可以输出到HBase完成存储，达到 HBase与Map Reduce协同工作。

**使用场景：**
1. 对HBase中的数据进行非实时性的统计分析。HBase适合做Key-Value查询，默认不带聚合函数（sum、avg等），对于这种需求非常适合集成Map Reduce来完成，但我们也应该注意到Map Reduce的局限性，Map Reduce本身的高延迟使得它不能满足实时交互式的计算。
2. 对HBase的表数据进行分布式计算。HBase的目标是在海量数据中快速定位所需的数据并访问它，可以发现HBase只能按照行键查询而不支持其他条件查询，所以我们只是依靠HBase来解决存储的扩展，而不是业务逻辑，那么此时将业务逻辑放到Map Reduce计算框架中是合适的。
3. 在多个Map Reduce间使用HBase作为中间存储介质。HBase在多个MapReduce作业中既是数据来源，又作为数据流向目的地。

MapReduce与HBase集成后，“输入/输出的文件”变为“表（HTable）”




