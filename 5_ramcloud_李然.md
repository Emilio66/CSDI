#RAMCloud（本人能力有限，如有错误请及时指出 wechat:elso002）
##助教说的考点

* 优缺点？
* 它的Recovery为什么是快的？
* 应该用多大的Segment进行配置，论文是怎么evaluation的？

##课前问题（后面解答）
* RAMCloud为什么使用log-structured策略？
* RAMCloud用了什么策略是放置Segment replicas，并且它在恢复的时候是怎么找到Segment relica的？
* RAMCloud是否支持随机访问？

##Motivation & Intro.
DRAM常被用作数据的缓存，但是Cache miss和backing store overhead都让DRAM的潜能无法激发出来。

-->因此论文作者设计了RAMCloud，它把所有的数据都置于DRAM中（这样不会出现Cache miss），又因为存储是durable的，所以
不需要额外地管理backing store。
  
RAMCloud的核心就是提供`durability & availability`

RAMCloud有一下几个特性：

* 利用scale: RAMCloud利用系统的大规模(large scale)来提供快速高效的错误恢复
* Log-structured storage: log-structured不仅用在了disk，也用在了DRAM。它为系统的提供了较高的性能
* 随机化(Randomization): RAMCloud使用随机化的策略来进行一些决策
* Tablet profiling: 为了实现快速恢复，它帮助划分服务器的数据到partition中

##数据模型
典型的key-value存储 --> 在RAMCloud中，数据被已表(table)的方式组织起来，表内存储的是大量的对象(object)。


RAMCloud管理replica是通过，在中DRAM只保存每个object的一个备份，而大量的冗余备份是位于二级存储中的（如disk或flash）。
通过这种方式备份会带来两个问题

* 将数据放入disk，必须要同步地对disk写入-->非常慢！
* 在宕机恢复的过程中，由于需要从二级存储读取数据来重构-->还是慢！

既然把数据备份在二级存储中慢，那备份到DRAM中呢？这就需要DRAM不能断电，而且会产生高昂的费用！
因此，RAMCloud还是选择把数据备份到二级存储中。

##Log-structured storage
上面提到了RAMCloud管理replica的策略，并且产生了两个问题。为了解决这两个问题，RAMCloud采用了log-structured的方式进行存储。

如上面图片所示，RAMCloud的存储的过程是当一个master接收到写请求，它将要写入的object append到内存里的log中，然后再将log entry通过网络分发到各个backup去。backup将接收到的信息buffer到内存中，然后立即向master返回一个RPC请求，而不需要立即写入到disk。master接收到backup的请求之后，向client返回。对于backup，在buffer满了之后才将buffer的数据写入disk。

可是考虑到一点，log是被buffer到backup的DRAM中的，那么一旦断电，就会导致数据的丢失---->为了解决这个问题，RAMCloud采用DIMM的方式，加入电池来保证buffer可以flush到disk上去。

注意到Hash table，hash table在RAMCloud是以<tableid, objectid>做key来定位内存中log的位置，在每次appen log的时候，都会在hash table建立索引--->hash table支持了对数据的随机访问

那么log-structured logging有如下特点：

* 由于backup是采用buffering的方式存储log（这是异步的、批量化的），这样避免了对disk的同步地写
* 只有在backup的buffer满了之后，才将数据flush到disk上，这样做就不会浪费大量的时间在等待磁盘写入
* log的结构是一致的，这句话的意思就是disk和DRAM的log结构一致，因此不需要额外的管理开销
* 使用了hash table实现了数据的随机访问

综上，log-structure的实现解决了RAMCloud管理replica的策略带来的第一个问题，并且它有***优点***在于充分利用sequential I/O带宽，避免了高延迟。而***缺点***在于，需要定期地对log进行cleaning操作。

##Recovery

由于RAMCloud是将数据存储在DRAM中，所以由于DRAM的容失性RAMCloud的错误恢复需要做到low latency。

###Using scaling
在论文中，argue了三种方法，分别是disk、CPU带来的bottleneck等等。
最Naive的想法就是，一台机器宕机之后，立刻从多个backup读取信息重新构建crash的节点。可是如果backup的数目过少，磁盘带宽就成为了瓶颈。那么，在上文中提到，RAMCloud充分利用了scale来实现快速recovery，既然backup数目少会让磁盘带宽成为瓶颈，那么增加backup势必会减轻disk带来的瓶颈。增加backup的数量之后，大量的数据通过网络抵达master节点，然后master节点再来replay。可是，当数据量变大之后，master的CPU性能就变成了瓶颈，这样也同样限制了master的网络接口。

而为了解决上述问题，RAMCloud采用了`Partitioned Recovery`策略，来进行错误恢复。思路也十分简单，RAMCloud将crashed master的object分割到若干recovery master上，让它们分开处理。并且，这些recovery master把这些数据整合到自己的log和hash table中。这样就解决了CPU/disk/network对性能带来的瓶颈，它的核心方法就是利用了系统的scaling。而这种方法的缺点就是需要与所有的host进行通信。

###Scatter Log segments

考虑到backup的热度，所处网络带宽，磁盘读写速度等问题，为了保证segment的放置尽可能的平均，RAMCloud采用了随机化(Randomization)和微调(refinement)相结合的方式来决定log segment的放置问题。它首先随机产生一个list，然后从总挑出一个最佳候选（根据磁盘读写速度等各个因素）。使用随机化，避免了多台master选择相同backup的情况，而随机化仍然会带来分配不均匀的情况，所以就需要微调来去解决这种情况。


###(后面的也没做笔记 -。- 大家看看论文自己理解吧)

对Recovery的总结：

* 利用scale去获得高可用性	--> 分散写来克服disk的bottleneck/分散rebuilding来克服CPU和网络的bottleneck
* Scale驱使了更低的latency

对于课前问题，1)在***Log-structured storage***这一章节已经给出答案；2)为啥是快的，因为他用了scaling啊(摊手) （大家自己总结吧 3)随机化，在log cleaning的时候用到了，在scatter log segment的时候用到了，还有在错误检测的时候用到了（错误检测的时候，RAMCloud需要定期的知道哪些节点故障了，它就随机地去ping一些机器）（这个我没写


##Evaluation

对于助教说的应该用多大的segment进行配置，臣妾没找到啊 :(
evaluation部分只说了partition大小的选取，过大的object在恢复的时候受到网络速度的限制，过小的object在恢复的时候受到更新hash table和tablet profile的限制。

对于segment的，论文在2.5节说到segment定位8MB大小。在evaluation部分，论文在论证系统可扩展性的时候，说道控制segment很小保证它全部都在内存中，然后partition的大小对scalability的影响（类似于控制变量法）。可以没有说对segment大小的evaluation - - (可能我没找到，找到的同学一定要告诉我



