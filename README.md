[toc]

![](./img/spark_tag.png)

_lighting-fast unified analytics engine_

# preface

用了这么长时间的spark一直没有静下心来总结一下这一段时间的学习和心得体会。这里整理成一个文档从浅到深来归纳总结，方便以后回顾时查阅方便

这个github将包含一系列用scala编写的demo, 数据集, spark的使用心得和踩坑，以及架构解释。如有错误，欢迎指正。和机器学习相关的特征预处理，模型训练，模型预测等模块会单独放到另一个github里来总结

GitHub地址： [https://github.com/Esail/sparkwiki](https://github.com/Esail/sparkwiki)



# spark overview

## what is spark?
spark官方给出的说明是，spark是一个**高效(light-fast)，统一(unified)，分析引擎(analytics engine)**
 
* 高效是因为spark是分布式的，并且基于内存来做计算.无论是在量还是在速上都有显著的提升
* 统一意味着它可以干任何和数据分析相关的工作，从sql查询到实时计算,再到机器学习，图计算都可以在以rdd为核心的spark引擎上完成。
* 分析引擎 spark是分析引擎，意味着它不负责数据的采集和数据的储存，你还需要依赖hdfs,hbase,cassandra来作为数据的储存介质，通过kafka,rabbitMQ等来收集日志。

下图展示了spark的组成：

![](./img/what_is_spark.png)


## spark architecture

这里简单的讲一下spark的结构，我们后面还会结合hadoop-yarn来具体讲一下。推荐所有人读Matei的**Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing**详细了解下spark的核心思想RDD的设计和实现


![](./img/driver.png)
<a name="来自matei的论文">来自matei的论文</a>

整个的架构分为driver和worker(executor)两个大的组成。从这个图上我们可以看出几个重要的信息：

* driver负责把任务tasks分发给worker来执行
* driver负责聚合worker执行task的results
* worker负责把数据拉到RAM（内存）中
* worker负责执行task计算任务

整个设计特别像是一台汽车，driver就是驾驶员，负责控制方向盘，监控仪表。而worker则是四个轮子，负责前行


## spark install
安装spark非常的简单，当然前提是你有一台mac电脑。
我们使用 brew轻松安装

```
brew update
brew install apache-spark
spark-shell --master local

```

这样会启动一个本地的spark交互界面，打开 [http://localhost:4040/jobs/](http://localhost:4040/jobs/) 可以看到spark的web界面

我们现在在交互界面完成第一个spark任务，不熟悉scala的同学可以去[https://www.tutorialspoint.com/scala/](https://www.tutorialspoint.com/scala/)快速熟悉一下语法。


```

1. 创建一个scala数组 1，2，3，4，5
scala> val testData = Array(1,2,3,4,5)
testData: Array[Int] = Array(1, 2, 3, 4, 5)

2. sc代表spark context，
3. parallelize的作用就是把一个local collection形成rdd
4. toDF()是指把rdd转化成DataFrame

scala> val testRDD = sc.parallelize(testData).toDF()
testRDD: org.apache.spark.sql.DataFrame = [value: int]

5. show()一下可以显示出来，我们看到spark把我们的数组默认成value
scala> testRDD.show()
+-----+
|value|
+-----+
|    1|
|    2|
|    3|
|    4|
|    5|
+-----+

6.可以通过 toDF("test")来改变这个名字

scala> val testRDD = sc.parallelize(testData).toDF("test")
testRDD: org.apache.spark.sql.DataFrame = [test: int]

scala> testRDD.show()
+----+
|test|
+----+
|   1|
|   2|
|   3|
|   4|
|   5|
+----+

```

上方是一个很简单的创建dataframe然后展示的操作。一个有意思的点是，我们创建的rdd到底是被切成了几份呢

通过testRDD.rdd.getNumPartitions可以看出数据被切成了一份。这个值由 defaultParallelism 来决定，而不同模式不同数据不同资源下这个值和分区数都是会变化的。我们暂时不关心这个值有什么用，但是后面spark调优时，这个值将发挥作用



# spark basic 
我们继续操作spark，在操作中熟悉spark的功能，这样稍后分析源码时会有一个直观的印象


## deal with text file
我们现在用spark操作一个文本文件

```
object TextDemo extends App {
	val spark = SparkSession.builder().master("local[*]").getOrCreate()
}
```

首先通过SparkSession来创建一个对话。这里分享一个小技巧，就是如何拿到resources下的文件路径

```
val words = this.getClass.getResource("/words.txt").getPath
```

如果想要在代码里将rdd转换成dataframe格式，需要

```
import spark.implicits._
```
隐式转换在spark是一个非常重要的概念，详情参考： https://docs.scala-lang.org/zh-cn/overviews/core/implicit-classes.html 

**完整代码**

```
import org.apache.spark.sql.SparkSession

object TextDemo extends App {
	val spark = SparkSession.builder().master("local[*]").getOrCreate()
	val words = this.getClass.getResource("/words.txt").getPath
	val sc = spark.sparkContext
	import spark.implicits._
	val lines = sc.textFile(words).toDF("word")
	lines.show()
}
```
可以看出全部显示成一列，然后列的title是 word

```
+---------+
|     word|
+---------+
|        A|
|        a|
|       aa|
```

我们统计个数 用count()

```
println(lines.count())
```

我们给所有词加个后缀

```
val newLines = lines.map(r => r.getAs[String]("word") + "_happy")
newLines.show()
```

通过map我们可以给集合中的每一个元素添加一个操作，而r=>r.getAs则可以拿出这个元素，然后添加后缀

如果预先已经知道了dataframe的列是如何分布的也可以通过r=>r.get(index)拿到对应的列，这样代码会更简洁，如下：

```
val newLines = lines.map(r => r.get(0) + "_happy")
newLines.show()
```

其实
val newLines = lines.map(r => r.get + "_happy") 竟然也可以工作，但是有一定副作用，这个留给大家去发现


### top n words
我们下面进行一个复杂的操作即经典的统计词频然后返回topN


```

object WordCountDemo extends App {
	val spark = SparkSession.builder().master("local[*]").getOrCreate()
	val words = this.getClass.getResource("/beauty.txt").getPath
	val sc = spark.sparkContext
	import spark.implicits._
	val wordsCount = sc.textFile(words)
		.flatMap(line => line.split(" "))
		.map(word => (word, 1))
		.reduceByKey(_+_)
		.toDF("word", "count")
		.orderBy($"count".desc)
	wordsCount.show()
	val n = 7
	val topN = wordsCount.take(n).mkString(",")
	println(topN)
}
```
输出

```
[the,32],[to,29],[and,27],[that,25],[of,22],[a,22],[was,17]
```

这里的flatMap, map, reduceByKey, orderBy都是spark强大的算子。spark提供了超过80种high-leverl operator来完成复杂的数据操作，关于算子的具体使用可以参考：

[https://spark.apache.org/docs/latest/rdd-programming-guide.html](https://spark.apache.org/docs/latest/rdd-programming-guide.html)



## transformation vs action
spark的操作主要分为transformation和action. 顾名思义，transformation只是做了变换，但并未真正执行。action操作时才会真正触发执行操作。action操作甚至支持异步的方式，这样执行不会阻塞


常见的transformation操作有：

* map(func): 对rdd的每个元素执行一个func操作，然后返回一个新的rdd
* fliter(func): 挑选elements即过滤掉不符合条件的元素
* flatMap(func):  压平操作，func这里返回的是seq而非单个item
* intersection: 交集操作
* groupByKey 聚集操作
* distinct 返回一个新的rdd包含去重后的元素

常见的actions有：

* reduce(func): 聚合元素
* collect(): Return all the elements of the dataset as an array at the driver program. This is usually useful after a filter or other operation that returns a sufficiently small subset of the data
* count(): 统计个数
* saveAsTextFile: 把rdd的elements写到hdfs或其他文件系统

具体每个的执行可以参考 operators这个目录。正是有了这些强大的算子，spark才会如此强大

## shuffle operations

shuffle顾名思义，洗牌，会重新分配数据(re-distribute data)，这样整体数据分布会发生变化(grouped differently across partitions)。 这个操作会造成数据在workers之间拷贝和传输(it involves disk I/O, data serialization, and network I/O)，是一个非常重的操作。实际使用中，我们应该尽量避免无必要的shuffle.



# Resilient Distributed Datasets

从这里开始我们会进入spark的核心部分，我强烈建议读者优先阅读官网的例子和github上一些基本操作.否则会看的很头疼


## what is rdd?
用paper中的表述就是，a distributed memory abstraction that lets programmers perform in-memory computations on large clusters in a fault-tolerant manner.
展开来说rdd即弹性数据集是：

* 一种数据抽象
* 用来在集群上进行基于内存的计算
* 不可变，可分区，里面的元素可并行计算
* 具备自动容错，位置感知性调度，可伸缩性等特点

![](./img/rdd.png)


我们看这个图来通俗理解rdd是什么，假设我们有8个单词需要操作，我开了4个分片(partition)，然后将这8个单词均分到4个分片上，最后每个节点上拥有两个分片。rdd就是整个分片的集合以及操作的抽象层（可以重点看下BlockPartition和BlockRDD这两个关系）


## rdd attributes

### partitions
rdd 拥有一组分片(partitions)，分片是rdd的基本组成单元。每个分片都会被分配一个计算任务(task)来处理。用户可以在创建rdd的时候制定rdd的分片个数，如果没有制定，将采用默认值。


### compute

```
 /**
   * :: DeveloperApi ::
   * Implemented by subclasses to compute a given partition.
   */
  @DeveloperApi
  def compute(split: Partition, context: TaskContext): Iterator[T]
```
用来在分片上执行计算，每个rdd的具体实现都要实现compute函数

### dependency

```
abstract class RDD[T: ClassTag](
    @transient private var _sc: SparkContext,
    @transient private var deps: Seq[Dependency[_]]
  ) extends Serializable with Logging {
```
rdd之间的依赖关系。rdd每次transform都会生成一个新的rdd,所以rdd之间就形成了依赖关系。如果部分分片数据丢失，spark可以通过这个依赖关系重新计算丢失分片数据，而不是对rdd的所有分片重新计算


### Partitioner
rdd的分片函数。一般用的有HashParitioner和RangePartitioner. 


### table
一个列表，存储存取每个Partition的优先位置（preferred location）。对于一个HDFS文件来说，这个列表保存的就是每个Partition所在的块的位置。按照“移动数据不如移动计算”的理念，Spark在进行任务调度的时候，会尽可能地将计算任务分配到其所要处理数据块的存储位置


## word count in picture

下图生动的表明了word count是如何在rdd之间传递的

![](./img/word_cnt.png)
![](./img/word_cnt2.png)


## wide dependency vs narrow dependency

宽依赖和宅依赖

简单来说窄依赖就是独生子女，又叫OneToOne dependency

```
/**
 * :: DeveloperApi ::
 * Base class for dependencies where each partition of the child RDD depends on a small number
 * of partitions of the parent RDD. Narrow dependencies allow for pipelined execution.
 */
@DeveloperApi
abstract class NarrowDependency[T](_rdd: RDD[T]) extends Dependency[T] {
  /**
   * Get the parent partitions for a child partition.
   * @param partitionId a partition of the child RDD
   * @return the partitions of the parent RDD that the child partition depends upon
   */
  def getParents(partitionId: Int): Seq[Int]

  override def rdd: RDD[T] = _rdd
}
```

代码如上图所示，宅依赖都会有一个getParents的method，由于是一对一的，这样恢复时只需要拿到其parent分区就可以。


宽依赖，又叫shuffle dependency

```
/**
 * :: DeveloperApi ::
 * Represents a dependency on the output of a shuffle stage. Note that in the case of shuffle,
 * the RDD is transient since we don't need it on the executor side.
 *
 * @param _rdd the parent RDD
 * @param partitioner partitioner used to partition the shuffle output
 * @param serializer [[org.apache.spark.serializer.Serializer Serializer]] to use. If not set
 *                   explicitly then the default serializer, as specified by `spark.serializer`
 *                   config option, will be used.
 * @param keyOrdering key ordering for RDD's shuffles
 * @param aggregator map/reduce-side aggregator for RDD's shuffle
 * @param mapSideCombine whether to perform partial aggregation (also known as map-side combine)
 */
@DeveloperApi
class ShuffleDependency[K: ClassTag, V: ClassTag, C: ClassTag](
    @transient private val _rdd: RDD[_ <: Product2[K, V]],
    val partitioner: Partitioner,
    val serializer: Serializer = SparkEnv.get.serializer,
    val keyOrdering: Option[Ordering[K]] = None,
    val aggregator: Option[Aggregator[K, V, C]] = None,
    val mapSideCombine: Boolean = false)
  extends Dependency[Product2[K, V]] {
```

很明显宽依赖要触发一次shuffle操作，其恢复是异常复杂的，下图中map,filter都属于宅依赖，而groupbyKey等都属于宽依赖

![](./img/dependency.png)

## DAG

![](./img/stage.png)


spark会根据rdd之间的依赖关系构建DAG（有向无环图）。而DAG具体会划分为不同的stage. 根据上图所示，由于宅依赖是一对一的，partition的转换处理可以在同一个线程里完成，宅依赖就被spark划分到同一个stage中。而对于宽依赖，只能等parenetRDD shuffle处理完成后，下一个stage才能开始计算

因此spark划分stage的整体思路是：从后往前推，遇到宽依赖就断开，划分为一个stage；遇到窄依赖就将这个RDD加入该stage中。因此在图2中RDD C,RDD D,RDD E,RDDF被构建在一个stage中,RDD A被构建在一个单独的Stage中,而RDD B和RDD G又被构建在同一个stage中。


## ShuffleMapTask vs ResultTask

在spark中，Task的类型分为2种：ShuffleMapTask和ResultTask；

简单来说，DAG的最后一个阶段会为每个结果的partition生成一个ResultTask，即每个Stage里面的Task的数量是由该Stage中最后一个RDD的Partition的数量所决定的！而其余所有阶段都会生成ShuffleMapTask；之所以称之为ShuffleMapTask是因为它需要将自己的计算结果通过shuffle到下一个stage中；也就是说上图中的stage1和stage2相当于mapreduce中的Mapper,而ResultTask所代表的stage3就相当于mapreduce中的reducer.


# advanced spark

## opeartion
### broadcast
### accumulators
## sparkcontext
## blockmanager
## netty in spark
## DAG Scheduler
## LauncherBackend



# how spark run


# tuning spark
spark 调优是理论结合实践的产物，是用好spark最好的参考


# spark sql


# spark memory management
spark对内存的管理

# spark tricks
这一章讲一些spark的小技巧


# spark on yarn
这一章讲hadoop yarn如何工作，以及spark如何在yarn上运行的


# spark streaming
这一章讲spark如何进行实时计算


# spark graphx
spark如何进行图计算

# extra spark
一些spark番外篇

# spark applications
spark应用

# advanced scala
scala高级语法篇

# spark machine learning
移步 spark-ml


# references:

## websites

[1] https://spark.apache.org/ (不用多说，官网是最好的文档，实践是最好的老师，源码是最佳的解释）

[2] https://www.cnblogs.com/qingyunzong/p/8886338.html (非常好的个人总结）

[3] https://www.jianshu.com/p/a2c2fe4371ea (hadoop3.0 new features)

## papers

[1] Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing (了解spark的诞生过程）


