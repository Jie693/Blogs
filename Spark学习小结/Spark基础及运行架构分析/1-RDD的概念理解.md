###RDD的概念理解

RDD是什么？



下面可以简单看下RDD的五个特性













>
弄清了上述概念，又看了网上各种讲解RDD的博客，仍然觉得RDD是个很抽象的概念，RDD具体是什么，在代码中是怎样呈现的依旧感觉很模糊。有很多博客直接将RDD解释为“一个分布式只读的大集合，将所有数据都加载到内存中，方便进行多次调用”，被这句话误导了很久，以至于后来看到cache和persist的时候更加想不明白了：既然RDD本身已经将所有数据加载到内存中，那何必需要额外设计cache这一步来把中间结果保存到内存中呢？

其实这种疑问完全来自于对RDD的误解，RDD实质上并不会把输入文件的所有数据都载入到内存中，RDD是一个逻辑概念，RDD更应该被看做是对“作用于数据集上的算子，数据集的依赖关系以及其他属性”的封装，并没有包含“物理数据”。RDD一般有两类：读取数据的RDD以及进行transformation的RDD

```
sc.textFile("xxx.txt").map().saveAsTextFile("")
```
上述代码的执行流程很清晰，textFile会构建出一个NewHadoopRDD（读数据的RDD）从HDFS上读取输入数据，然后map函数产生一个MappedRDD（进行transformation的RDD）对上一个RDD进行突然是transformation，最后saveAsTextFile会触发job，并将生成的数据输出到HDFS上。但是具体在内存中到底载入了多少数据呢？如果把所有的输入数据都载入到内存中，如何对所有数据进行transformation呢？

实际上看一下RDD的源代码（另一篇笔记会详细讲解RDD源码[RDD源码解析](../Spark源码分析/RDD源码解析) ),可以看到有一个重要的方法：compute
> from Class RDD

```
  /**
   * :: DeveloperApi ::
   * Implemented by subclasses to compute a given partition.
   */
  @DeveloperApi
  def compute(split: Partition, context: TaskContext): Iterator[T]
```
该方法会为partition返回一个Iterator，以便逐条计算访问partition的数据。
而实际上对于两类不同的RDD，compute方法的具体实现也有两种：  
1、没有前置依赖的RDD，如读取数据的RDD（eg:NewHadoopRDD）的compute方法，主要功能为根据分片的信息生成遍历数据的Iterable接口

2、有前置依赖的RDD，如transformation的RDD（eg: MappedRDD)的compute方法，主要功能为在parent RDD的iterator接口上调用transformation算子，即在parent RDD遍历数据时套用新的transformaton算子计算
> from Class MappedRDD

```
 override def compute(split: Partition, context: TaskContext) =
    firstParent[T].iterator(split, context).map(f)
```

所以实际上，在内存中下面代码的执行流程为，当NewHadoopRDD遍历数据时，每加载一条数据，该数据就会在内存中实现map的transformation，随后存入HDFS中，也就是说理想情况下，集群内存中只用保存每个partition的某一条数据即可（假设集群只用跑一轮就可以跑完任务，即所有partition同时在被执行），总内存大小占用为partitionNum * SizePerRecord。  
然而，事实上和理想情况还是有所区别，特别是对NewHadoopRDD以及saveAsTextFile，因为这两个步骤涉及读写磁盘，不可能一条条地从磁盘读写数据，因此一般会存在一个buffer。

```
sc.textFile("xxx.txt").map().saveAsTextFile("")
```
我们常常说的Spark是内存型的数据处理引擎，其实并不是指RDD将所有的数据载入到内存中所以计算速度快，实际上如果没有主动调用cache, RDD并不会将所有数据加载到内存中。而Spark之所以运行速度快很大程度上是由于其优秀的调度机制（DAGScheduler等）以及容错（比如使用cache之后，中间数据会cache到内存中，后面的计算如果出错，可以利用内存中cache的中间数据重新计算失败的stage，而不用像MR一样返回第一步重头开始）。

然而这又引出了另一个问题：既然RDD不会把所有数据加载到内存中，为什么说Spark是基于内存的数据处理引擎？  
我个人感觉“基于内存"这一点主要是要和MapReduce来进行比较，主要基于两点原因：
1、对RDD进行cache或persist可以将常用的RDD cache到内存中，这样方便重复利用以及错误恢复  
2、算子的pipeline: 对于MapReduce，任何计算Map和Reduce之间的中间数据都必须落盘（本地盘）。 而Spark可以利用DAG来理性划分stage，且stage内部算子与算子之间数据不会落盘。简而言之，Spark可以依照宽依赖来划分stage，而stage内部的算子之间的关系为窄依赖。而MapReduce划分Map和Reduce并不考虑宽窄依赖（宽依赖一定分开为Map和Reduce，而窄依赖也可能被划分开），这就导致了窄依赖的也可能需要落盘，从而降低效率和灵活性。



References:  
[http://www.jianshu.com/p/b70fe63a77a8/comments/2140231](http://www.jianshu.com/p/b70fe63a77a8/comments/2140231)