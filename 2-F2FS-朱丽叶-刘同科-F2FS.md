#flash的技术特性（背景了解：为什么不用传统的数据库在flash上写）

-flash的读/写和擦除操作是不对称的（读/写是以页为单位，擦除是以块为单位）

-写前必须擦除，每个块的擦除操作是有一定寿命的。

#六个技术点,

-Flash-friendly on-disk layout:文件系统分为superblock(SB),Checkpoint(CP),Segment Info(SIT),Node Address

Table(NAT),Segment Summary Area(SSA),和Main Area六个部分存储,在分配存储块的时候用块做单位,cleaning的时候用section做单位，与FTL操作单元对齐，减少不必要的数据拷贝。

-Cost-effective index structure:解决了wandering tree的问题。

-multi-head logging:有多个log区域，有助于数据冷热分离。

-adaptive logging:使用append-only

logging让随机写变顺序写，在内存利用率较高的时候，进行thread logging。

-cleaning:Foreground cleaning(空间不足自动触发)background cleaning(周期执行)

选择vitcim(greed(FC),cost-benefit(BC))->有效块的确认和迁移(通过SIT确认有效块,然后把块放到page cache里面标记为dirty,不会发送I/O)->发送清理进程(victim section被标记为pre-free section,checkpoint做了后,标记为free)

-Fsync acceleration with roll-forward recovery:对small writes进行优化，降低fsync的延迟(使用specicial降低checkpoint开销)

#checkpoint怎么做

1. flush所有的脏节点和目录项在page cache

2. 它将挂起写活动，包括系统调用如create,unlinkandmkdir;

3. 文件系统的元数据，NAT，SIT和SSA被写入磁盘中特定区域

4. 写一个包括以下信息的checkpoint pack到CP区域(Header and

footer/NAT and SIT bitmaps/NAT and SIT journals/Summary blocks of active

segments/Orphan blocks)

#Wandering tree问题

-wandering tree问题是log-structured文件系统（LFS）特有的一个问题，因为LFS的脏数据是追加更新的，所以如果一个数据块变脏了，那么那个数据块的直接索引块、间接索引块都会变脏（因为索引的地址变脏）。

-在F2FS中使用NAT表来存储node id和实际物理地址的映射关系，这样一个数据块变脏的时候，只需要更新直接节点块和它的NAT表项。

#Why does F2FS try to reducethe effects of random write? What techniques does it use?

1.随机写写入的性能差，因为块擦除时间在ms级；

2.不断的对同一Block块进行擦除操作，那么该块将会在短时间内磨损写坏，并且极易导致存储在该块上的数据丢失。

Adaptive logging中的normal logging(append only logging),能将随机写变成顺序写

#What is 'wandering tree'problem? How does F2FS mitigate it?

见上

#What is the design rationale of multi-head logging? How does F2FS separate cold/hot data?

F2FS维护六个主要日志区域来进行冷热数据的分离，冷热数据有三个温度(hot/warm/cold)，目录项的node和data都是hot，文件直接节点和用户数据是warm，间接节点和cleanning清掉的数据/用户标记的冷数据/多媒体文件都是cold。通过预期的更新频率进行冷热数据的分离。

##roll-forward的问题，说一下自己的理解。
首先roll-forward recovery是为了解决经常写一些小数据，引发fsync操作，然后支持fsync操作的一个比较差的方法是做cp，然后回滚。因为cp会产生把所有节点和无关项之类的都写一遍。造成性能问题。
但是做cp和回滚肯定是要做的，为了改进性能，对一些小的写操作，F2FS在他们直接的节点块里面设立一个特殊的flag。这样就可以找到这个写操作。
在实际recovery操作中，F2FS收集在上一个有效cp后的对某个文件的写操作，可能是cp后的n个操作，记为N+n，N就是做cp的点。
然后找到在做这个cp之前的对这个文件的写操作，因为这个写操作做了cp，所以是一个稳定的写入磁盘的点。记为N-n。
比较cp前和cp后，N+n和N-n的不同，如果不同，使用N+n，刷新了N-n在cp的稳定存储，就可以跳过cp后一系列的操作，直接进行small write的更新，从而进行优化。
概括一下。。。。。。。。。。。。。。。。。。分割线
就是N+n是做了cp后这个数据更新的点，因为有特殊的flag，所以我们可以找到它，记为N+n。然后recovery的时候，使用cp，cp有个稳定的记录，能得到这个数据在cp之前的稳定的状态，记为N-n，进行对比，不同就使用最新的那个。

我个人的理解，有问题欢迎拍砖。最近很多人问这个问题，不过这个应该不是重点。。。

