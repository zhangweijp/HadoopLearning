# MapReduce2.0 On Yarn
MapReduce是Hadoop的核心组件中的分布式并行编程框架，顾名思义，MapReduce主要做以下两个工作：
1. Map：负责数据的映射，将数据做成键值对；
2. Reduce：根据映射的结果对数据进行整合。  
## MapReduce的逻辑架构
### Split(切片)
Map操作首先是将整个文件在**逻辑形式**进行切分(block是物理上的切分)，得到一个个的Split，为什么要切分呢？因为我们要多个Mapper并行分布式处理来提升计算的效率，每一个Split都会由一个且仅由一个Mapper来处理(一对一的关系)，比如一个文件被切分成了N个Split，那么就会有N个Mapper来并行处理这些Split。  
Split的默认大小与Block相同，但也不会被此局限，你可以根据实际需求在系统所规定的范围内调节Split的大小，也就是说可以一个Block对应多个Split，或者多个Block对应一个Split，值得注意的是，这里所有的Block和所有的Split都同属于一个文件。
### Mapper
MapTask将读取Split中的**一条条记录**，**每一条记录都会调用一次**Mapper方法(Reducer方法是**一组数据调用一次**)，记录就被转化成<K,V>**键值对**，之后会触发一个算法给每个<K,V>添加一个**分区号P**，P规定了该<K,V>归属于哪个Partiton(Partition和ReduceTask之间的一一对应的关系)，即计算出了该<K,V>在Reduce过程中归属于哪个ReduceTask；随后我们在内存中对<K,V>-P进行分类，首先按照分区号P进行外部排序，然后再按照Key值在分区内进行排序，这样数据就按照**外部按分区，内部按分组**的方式进行了分类；因为我们每次是抽取一部分数据在**内存中进行排序**，排序之后的小文件存放在硬盘上(一次IO)，所以最后还要在**硬盘**内将这些数据**归并(Merge)**和**整合(combine)**，整合(可以减少数据量，节省网络传输和I/O的开销)完毕之后，再按照分区号将数据发送给相应的ReduceTask(Shuffle)。  
**为什么需要排序？**  
我们知道Reducer对数据的操作是按组进行的(**Key值相同的键值对为一组数据**)
- 如果Map完以后不对键值对进行排序，那么，ReduceTask收到的就是乱序的键值对的文件，就意味着，Reducer每次都需要遍历整个文件才能从中提取出一组数据，分组越多，Reducer遍历整个文件的次数就越多，花费的时间也就越长，效率就低下；
- 如果我们在Map的时候就对键值对进行排序，Reducer收到的就是一个个按照分组排好序的文件，Reducer就可以使用归并(merge)将其整合成更完整的文件(Reducer的排序强依赖于Mapper排序输出的结果)，可以节省下大量的时间，且MapTask中Mapper的排序是并行进行的，也是高效的。
### Partition
一个Reducer是可以处理一组或者多组数据，但是一组数据只能被一个Reducer处理，所以Reducer与分组数据之间的关系是一对多(1->N)，所以Partition中是可以存放一组或者多组数据(有多个Key值)，但一组数据只能有一个的分区号；
### Reducer
ReduceTask收到了来自不同Mapper但处于同一Partition的分块，将它们进行归并(Merge)整合(combie)，一边归并一边计算，得到该分组的结果并输出。
## MapReduce2.0 On Yarn 的运行架构
在Hadoop2.x中MapReduce、Spark等计算框架是运行在资源调度框架Yarn之上的。
- ResourceManager(Yarn)  
  ResourceManage管理整个集群的资源调度。  
  ResourceManager负责管理集群中的NodeManager，，当有Client对ResourceManager提交任务申请时，ResourceManager会根据NodeManager反馈的信息，选择合适的节点上启动和管理ApplicationMaster来执行任务，若ApplicationMaster运行失败，ResourceManager会对其重新启动。
- NodeManager(Yarn)  
  每一个DataNode上都有一个NodeManager，NodeManager会与ResourceManager保持通信，汇报本节点的资源使用情况，通过对Container的管理来实现对本节点资源的管理，主要负责管理每个Container的生命周期、监控每个Container的资源使用情况、追踪节点资源(Cpu、内存)使用情况等。
- Container  
  task的容器，Container将内存、 CPU、磁盘、网络等资源封装在一起，这样可以起到限定资源边界的作用。
- ApplicationMaster  
  ApplicationMaster被ResourceManager启动后，会从HDFS获取文件资源的分片信息，将其发送给ResourceManager，ResourceManager将分配Container给Application使用，Application监控整个任务的执行，跟踪整个任务的状态，处理任务失败以异常情况。