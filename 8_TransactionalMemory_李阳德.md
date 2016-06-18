<link rel="stylesheet" href="http://yandex.st/highlightjs/6.1/styles/default.min.css">
<script src="http://yandex.st/highlightjs/6.1/highlight.min.js"></script>
<script>
hljs.tabReplace = ' ';
hljs.initHighlightingOnLoad();
</script>
# Transactional Memory/DB,Transactional memory: architectural support for lock-free data structures
## 助教提供的考点

---

1. 与传统锁的区别
2. 为什么要引入Transaction Memory
3. 具体的Transaction memory有哪些技术，__熟悉技术细节__

---

## 关键科学问题
多核处理器越来越普及，如何利用硬件性能，更加高效的提升并行程序的性能成为难点。基于锁的同步机制存在以下问题：

* 粗粒度的锁虽然易于使用，但是性能低下，锁竞争频繁，无关的操作会出现顺序执行的情况
* 细粒度的锁虽然性能高，但是实现难度大，例如双端队列的并行版本使用细粒度的锁难以实现
* 存在优先级反转（priority inversion）、护航（conveying）、死锁（dead lock）等问题
    * _优先级反转_ 一个高优先级任务间接被一个低优先级任务所抢先(preemtped)，使得两个任务的相对优先级被倒置
    * _护航_ 多线程程序中相同优先级任务反复竞争同一个锁。
    * _死锁_ 两个或以上任务在执行过程中，由于竞争资源造成阻塞    
* 可组合型差。例如从一个链表移动一个元素到另一个链表时，必须使用低性能的粗粒度的锁来避免可能发生的死锁 



---

## 解决方法
新的同步机制==> __无锁算法__、__无锁的数据机构__

参考数据库系统中的事务的概念，引入事务的ACID概念。本篇论文介绍事务性内存--一种新的多核架构，使用无锁的同步机制并使其与传统锁机制下达到同样的性能并保持简单。

可以使用软件来模拟事务执行，也可以使用硬件来加速支持（硬件更加可靠？）

* STM: Software Transactional Memory
* HTM: Hardware Transactional Memory 

intel Haswell 使用了Restricted Transactional Memory (RTM)，一种受限制的硬件事务内存实现。新增了三个
指令如下所示,但是work set是受限的，一些System event废弃了TX

```
Xbegin
Xend
Xabort

```


---

### 技术细节

事务是有限的机器指令序列，被单个进程所执行，并满足有序性和原子两个特性。有序性意味着一个事务执行的步骤不会和另一个事务相交错。原子性指每个事务都会暂时的修改共享内存，当一个事务完成后，要么commit，使修改对其他进程可见；要么abort，废弃自己的修改，不存在其他的中间状态。



#### 基本指令

* 访问内存的基本指令

```

Load-transactional LT //从共享内存出读取值到私有寄存器

Load-transactional exclusive LTX // 从共享内存出读取值到私有寄存器并标记 *即将更新*

Store-transactional ST //将私有寄存器中的值写入共享内存中（write set），尚不对外可见


```

一个事务的read set是指被LT指令读取的位置，它的write set指被 LTX 或者 ST 指令访问的位置。read和write组成了事务的data set.


* 修改状态的基本指令

```
Commit(COMMIT) //尝试使事务所做的修改持久化,其他事务不再更新本事务的data set，没有别的事务*读取过*本事务的write set，即commit成功，并使该事务对于write set所做的修改对别的事务可见。

Abort(ABORT)   // 取消所有对于write set的临时修改。

Validate(VALIDATE) // 测试当前事务的状态，返回true说明当前还没有abort执行，返回false说明当前事务已经被abort

```

通过对以上指令的组合和使用，开发者可以实现自己的一套read-modify-write操作。本文所实现的transacnal memory访问指令与原有的指令并不冲突，也兼容原有的 `LOAD`、`STORE`指令。



### 基本原理

* 具体执行流程

![img](/img/8_1.png)

1. 使用`LT`或者`LTX`指令从共享内存中读取数据
2. 使用`VALIDATE`指令确认读取数据的有效性
3. 使用`ST`指令来修改共享内存中的数据
4. 使用`COMMIT`指令使修改生效；如果`VALIDATE`或者`COMMIT`失败，返回步骤1

### 具体实现

本论文基于多核缓存一致性协议进行了扩展，实现对事务性内存的支持。具体的协议包括：

* 共享总线（Snoopy cache）
* 基于目录（directory）

两种结构的支持。任何具有冲突访问检测能力的协议都可以用来检测事务冲突，不需要带来额外的开销。所以本文的实现直接复用了原有的协议。

__以共享总线结构的协议为例:__

#### 缓存

为了最小化对非事务性内存指令影响，每个处理器持有两种cache：

* regular cache （非事务性内存）
* transactional cache (事务性内存)

这两种cache是互斥的，一次访问只可能访问一种cache，而且这两种cache都是一级cache或者二级cache，可以被处理器直接访问。regular cache是传统的直接映射cache；transactional cache空间较小，完全相连的由额外的逻辑来实现事务的提交和废弃。事务cache存储所有临时写的副本，除非事务提交，否则缓存内容将不会传给其他处理器或内存。

遵循`Goodman`大神的指示，每个cache行的状态如下表所示

| Name       | Access          | Shared?  | Modified?|
| :-------------: |:-------------:| :-----:|:-----:|
| INVALID     | none |  |    |
| VALID     | R     |   Yes |  Yes  |
| DIRTY | R,W    |    No |     Yes   |
| RESERVED | R,W      |    No |    No    |


但是这些状态不够描述事务性内存访问，所以我们扩充了一些状态来描述事务内存的执行状态

| Name|Meaning|
|:----:|:----:|
|EMPTY| contains no data|
|NORMAL| contains committed data|
|XCOMMIT|discard on commit|
|XABORT|discard on abort|

事务性操作缓存有两个标志位，分别为`XCOMMIT`，`XABORT`。当事务提交时，会将`XCOMMIT`标记为`EMPTY`，将`XABORT`标记为`NORMAL`；当事务废弃时，会将`XABORT`标记为`EMPTY`，`XCOMMIT`标记为`NORMAL`
当事务性缓存需要空间时，首先搜索被标记为`EMPTY`的位置，之后再搜索被标记为`NORMAL`的位置，最后才是`XCOMMIT`的位置，由此来确定优先级，并且避免访问竞争资源提升性能。

#### 总线

除了缓存状态，对于共享总线结构的协议来说，还有总线周期，具体见下表

| Name       | Kind          | Meaning  | New access|
| :-------------: |:-------------:| :-----:|:-----:|
| READ     | regular |  read value|  shared  |
| RFO     | regular     |   read value |  exclusive  |
| WRITE | both    |    write back |     exclusive   |
| T_READ | trans  |    read value |    shared    |
| T_RFO | trans      |    read value |    exclusive    |
| BUSY | trans      |    refuse access |    unchanged    |

以上的cycle中前三个是`Goodman`大大协议总已经定义过了的，本文扩展了三个周期，`T_READ`、`T_RFO`和`BUSY`，前两个就是对原有周期的简单扩展，`BUSY`周期是为了防止各个transation之间过于频繁的互相abort而设立的，当食物接收到BUSY回应后，会立即abort并retry，理论上防止了资源饥饿。

#### 处理器
每个处理器持有两个状态标识位：

```
TACTIVE(transaction active)//是否有事务在运行
TSTATUS(transaction status)//事务是否是active还是aborted

```

**注**：`TACTIVE`标识会在事务在执行第一次事务操作时被隐式（偷偷地）设定。

---

一个被标记为`TSTATUS`为TRUE的事务LT指令的具体操作流程：

1. 探测事务缓存中的`XABORT`入口，如果有则返回值
2. 如果没有但是有`NORMAL`入口，将`NORMAL`改为`XABORT`入口，使用相同的标记`XCOMMIT`和相同的数据再分配一个入口
3. 如果没有`XABORT`或`NORMAL`入口，进入`T_READ`周期，如果成功，我们会设置两个事务性内存的入口：`XCOMMIT`、`XABORT`。每个都满足之前的协议，并进入到`READ`周期
4. 如果收到`BUSY`信号，则abort事务

对于LTX指令，我们使用T_RFO周期取代T_READ，如果上一周期成功就将cache line的状态标记为RESERVED。ST指令类似LTX，只是更新了XABORT入口的数据。cache line的标记，LT、LTX指令类似LOAD，ST类似STORE。VALIDATE指令返回TSTATUS状态，如果返回false，则将TACTIVE标志指定为false，并将TSTATUS指定为ture。ABORT指令会忽略cache入口，将TSTATUS置为true，TACTIVE置为false。COMMIT指令会返回TSTATUS，设置TSTATUS为true，TACTIVE为false，放弃所有的XCOMMIT缓存入口，并将所有XABORT标志变为NORMAL。最后，遇到终端和缓存溢出则会abort当前事务
 


---


## 课后题解答

---

> What is lock-free data structure? Why do we need lock-free data structure?
> 什么是无锁的数据结构，为什么我们需要无锁的数据结构？

无锁的数据结构是一种能够`无需使用锁`就能支持`多重读写`操作的数据结构。使用无锁的数据结构我们能够避免大量因为维持全局锁操作所带来的额外开销。

---

> What is orphan transaction? Why do we need VALIDATE instruction?
> 什么是孤儿事务？为什么我们需要VALIDATE指令？

__孤儿事务__ 是指在被废弃（aborted）之后继续执行的事务（例如，在另一个已经提交过的事务更新了自己的read set之后）
事务在执行过程中利用`VALIDATE`指令来确保自己所读取的值是正确的，防止读取过期的数据，即保证了数据的一致性。

---


> Open question: What is the shortcoming of transactional memory compared with 
>  conventional locking techniques? Do you have any suggestion to mitigate those 
>  problems?
>  开放问题： 事务性内存相比于传统锁技术有哪些缺点? 你对这些缺点有哪些解决方案？

**局限性**

* 事务性内存是基于事务具有短周期和小的数据集的假定。如果一个事务运行地越久，它就越有可能被中断或同步冲突所中止。同样地，数据集越大，其所需的事务性缓存也就越大，也就使得同步冲突发生的可能性越大。
* 本实现过程不支持将冲突的事务提前处理，而是依赖于软件层面上的以加大冲突事务的间隔时间的方式来自适应的后移事务以减少事务的中止比例。
* 本实验中的仿真所采用的缓存一致性协议提供了顺序一致性内存。但有一些研究者提出采用更加弱化的一致性协议以期达到更加高效的实现。如处理器一致性(Processor consistency)、弱一致性(week consistency)、释放一致性(release consistency)等等。

**解决方案**

* 为了支持更大更长的事务，可以采用更加复杂且精心设计的硬件机制；
* 为了降低事务中止的比例，可以采用硬件上的排队机制；
* 为了达到更加高效地实现，可以采用弱化的缓存一致性协议。如为了保障内存的顺序一致性，编程人员可以在每一个临界区的开始和结束时执行一个屏障指令(barrier instruction)。为提供事务性内存语义上的弱一致性内存，最直接的方式是使每一个事务性指令执行一个隐性障碍(implicit barrier)。









