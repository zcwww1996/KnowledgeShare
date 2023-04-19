[TOC]
# 1. spark.shuffle.manager
Spark 1.2.0官方支持两种方式的Shuffle，即Hash Based Shuffle和Sort Based Shuffle。其中在Spark 1.0之前仅支持Hash Based Shuffle。Spark 1.1的时候引入了Sort Based Shuffle。Spark 1.2的默认Shuffle机制从Hash变成了Sort。如果需要Hash Based Shuffle，可以将spark.shuffle.manager设置成“hash”即可。

如果对性能有比较苛刻的要求，那么就要理解这两种不同的Shuffle机制的原理，结合具体的应用场景进行选择。

## 1.1 Hash Based Shuffle
Hash Based Shuffle，就是将数据根据Hash的结果，将各个Reducer partition的数据写到单独的文件中去，写数据时不会有排序的操作。这个问题就是如果Reducer的partition比较多的时候，会产生大量的磁盘文件。这会带来两个问题：

1. 同时打开的文件比较多，那么大量的文件句柄和写操作分配的临时内存会非常大，对于内存的使用和GC带来很多的压力。尤其是在Sparkon YARN的模式下，Executor分配的内存普遍比较小的时候，这个问题会更严重。
2. 从整体来看，这些文件带来大量的随机读，读性能可能会遇到瓶颈。

## 1.2 Sort Based Shuffle
Sort Based Shuffle会根据实际情况对数据采用不同的方式进行Sort。这个排序可能仅仅是按照Reducer的partition进行排序，保证同一个Shuffle Map Task的对应于不同的Reducer的partition的数据都可以写到同一个数据文件，通过一个Offset来标记不同的Reducer partition的分界。因此一个Shuffle Map Task仅仅会生成一个数据文件（还有一个index索引文件），从而避免了Hash Based Shuffle文件数量过多的问题。

选择Hash还是Sort，取决于内存，排序和文件操作等因素的综合影响。

对于不需要进行排序的Shuffle而且Shuffle产生的文件数量不是特别多，Hash Based Shuffle可能是个更好的选择；毕竟Sort Based Shuffle至少会按照Reducer的partition进行排序。

而Sort BasedShuffle的优势就在于Scalability，它的出现实际上很大程度上是解决Hash Based Shuffle的Scalability的问题。由于Sort Based Shuffle还在不断的演进中，因此Sort Based Shuffle的性能会得到不断的改善。

对选择那种Shuffle，如果对于性能要求苛刻，最好还是通过实际的场景中测试后再决定。不过选择默认的Sort，可以满足大部分的场景需要。

# 2. spark.shuffle.spill
这个参数的默认值是true，用于指定Shuffle过程中如果内存中的数据超过阈值（参考spark.shuffle.memoryFraction的设置），那么是否需要将部分数据临时写入外部存储。如果设置为false，那么这个过程就会一直使用内存，会有Out Of Memory的风险。因此只有在确定内存足够使用时，才可以将这个选项设置为false。

对于Hash BasedShuffle的Shuffle Write过程中使用的org.apache.spark.util.collection.AppendOnlyMap就是全内存的方式，而org.apache.spark.util.collection.ExternalAppendOnlyMap对org.apache.spark.util.collection.AppendOnlyMap有了进一步的封装，在内存使用超过阈值时会将它spill到外部存储，在最后的时候会对这些临时文件进行Merge。

而Sort BasedShuffle Write使用到的org.apache.spark.util.collection.ExternalSorter也会有类似的spill。

而对于ShuffleRead，如果需要做aggregate，也可能在aggregate的过程中将数据spill的外部存储。

# 3. spark.shuffle.memoryFraction和spark.shuffle.safetyFraction
在启用spark.shuffle.spill的情况下，spark.shuffle.memoryFraction决定了当Shuffle过程中使用的内存达到总内存多少比例的时候开始Spill。在Spark 1.2.0里，这个值是0.2。通过这个参数可以设置Shuffle过程占用内存的大小，它直接影响了Spill的频率和GC。

 如果Spill的频率太高，那么可以适当的增加spark.shuffle.memoryFraction来增加Shuffle过程的可用内存数，进而减少Spill的频率。当然为了避免OOM（内存溢出），可能就需要减少RDD cache所用的内存，即需要减少spark.storage.memoryFraction的值；但是减少RDD cache所用的内存有可能会带来其他的影响，因此需要综合考量。

在Shuffle过程中，Shuffle占用的内存数是估计出来的，并不是每次新增的数据项都会计算一次占用的内存大小，这样做是为了降低时间开销。但是估计也会有误差，因此存在实际使用的内存数比估算值要大的情况，因此参数 spark.shuffle.safetyFraction作为一个保险系数降低实际Shuffle过程所需要的内存值，降低实际内存超出用户配置值的风险。

# 4. spark.shuffle.sort.bypassMergeThreshold
这个配置的默认值是200，用于设置在Reducer的partition数目少于多少的时候，Sort Based Shuffle内部不使用Merge Sort的方式处理数据，而是直接将每个partition写入单独的文件。这个方式和Hash Based的方式是类似的，区别就是在最后这些文件还是会合并成一个单独的文件，并通过一个index索引文件来标记不同partition的位置信息。从Reducer看来，数据文件和索引文件的格式和内部是否做过Merge Sort是完全相同的。

这个可以看做SortBased Shuffle在Shuffle量比较小的时候对于Hash Based Shuffle的一种折衷。当然了它和Hash Based Shuffle一样，也存在同时打开文件过多导致内存占用增加的问题。因此如果GC比较严重或者内存比较紧张，可以适当的降低这个值。

# 5. spark.shuffle.blockTransferService
在Spark 1.2.0，这个配置的默认值是netty，而之前是nio。这个主要是用于在各个Executor之间传输Shuffle数据。Netty的实现更加简洁，但实际上用户不用太关心这个选项。除非是有特殊的需求，否则采用默认配置就可以。

# 6. spark.shuffle.consolidateFiles
这个配置的默认配置是false。主要是为了解决在Hash Based Shuffle过程中产生过多文件的问题。如果配置选项为true，那么对于同一个Core上运行的Shuffle Map Task不会新产生一个Shuffle文件而是重用原来的。但是每个Shuffle Map Task还是需要产生下游Task数量的文件，因此它并没有减少同时打开文件的数量。如果需要了解更加详细的细节，可以阅读7.1节。

但是consolidateFiles的机制在Spark 0.8.1就引入了，到Spark 1.2.0还是没有稳定下来。从源码实现的角度看，实现源码是非常简单的，但是由于涉及本地的文件系统等限制，这个策略可能会带来各种各样的问题。由于它并没有减少同时打开文件的数量，因此不能减少由文件句柄带来的内存消耗。如果面临Shuffle的文件数量非常大，那么是否打开这个选项最好还是通过实际测试后再决定。

# 7. spark.shuffle.service.enabled
(false)

# 8. spark.shuffle.compress和 spark.shuffle.spill.compress
 这两个参数的默认配置都是true。spark.shuffle.compress和spark.shuffle.spill.compress都是用来设置Shuffle过程中是否对Shuffle数据进行压缩；其中前者针对最终写入本地文件系统的输出文件，后者针对在处理过程需要spill到外部存储的中间数据。

## 8.1 如何设置spark.shuffle.compress?

如果下游的Task通过网络获取上游Shuffle Map Task的结果的网络IO成为瓶颈，那么就需要考虑将它设置为true：通过压缩数据来减少网络IO。由于上游Shuffle Map Task和下游的Task现阶段是不会并行处理的，即上游Shuffle Map Task处理完成，然后下游的Task才会开始执行。因此如果需要压缩的时间消耗就是Shuffle MapTask压缩数据的时间 + 网络传输的时间 + 下游Task解压的时间；而不需要压缩的时间消耗仅仅是网络传输的时间。因此需要评估压缩解压时间带来的时间消耗和因为数据压缩带来的时间节省。如果网络成为瓶颈，比如集群普遍使用的是千兆网络，那么可能将这个选项设置为true是合理的；如果计算是CPU密集型的，那么可能将这个选项设置为false才更好。

## 8.2 如何设置spark.shuffle.spill.compress？

如果设置为true，代表处理的中间结果在spill到本地硬盘时都会进行压缩，在将中间结果取回进行merge的时候，要进行解压。因此要综合考虑CPU由于引入压缩解压的消耗时间和Disk IO因为压缩带来的节省时间的比较。在Disk IO成为瓶颈的场景下，这个被设置为true可能比较合适；如果本地硬盘是SSD，那么这个设置为false可能比较合适。

# 9. spark.reducer.maxMbInFlight
这个参数用于限制一个ReducerTask向其他的Executor请求Shuffle数据时所占用的最大内存数，尤其是如果网卡是千兆和千兆以下的网卡时。默认值是48MB。设置这个值需要中和考虑网卡带宽和内存。