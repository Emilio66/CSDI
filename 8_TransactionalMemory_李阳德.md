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

可以使用软件

STM: Software Transactional Memory
HTM: Hardware Transactional Memory 


---

### 技术细节

事务是有限的机器指令序列，被单个进程所执行，并满足有序性和原子两个特性。有序性意味着一个事务执行的步骤不会和另一个事务相交错。原子性指每个事务都会暂时的修改共享内存，当一个事务完成后，要么commit，使修改对其他进程可见；要么abort，废弃自己的修改，不存在其他的中间状态。



#### 基本指令

* 访问内存的基本指令

```

Load-transactional LT //从共享内存出读取值到私有寄存器

Load-transactional exclusive LTX // 从共享内存出读取值到私有寄存器并标记 **即将更新**

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

具体执行流程

![img](/img/8_1.png)

### 仿真代码

``` c

```
 

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


---


> Open question: What is the shortcoming of transactional memory compared with 
>  conventional locking techniques? Do you have any suggestion to mitigate those 
>  problems?
>  开放问题： 事务性内存相比于传统锁技术有哪些缺点? 你对这些缺点有哪些解决方案？

__TBD__





