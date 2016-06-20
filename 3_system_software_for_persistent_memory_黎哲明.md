#System Software for Persistent Memory##助教提供的考点
1. NVM技术特性跟flash有什么不同  2. consistency重点【Copy on write/journaling/log-structured】 3. pm_w_barrier为什么引入  CoLFLUSH 4. reoder 3个指令引入原因，指令作用，相互关系
###概述：
PMFS利用PM的byte-addressability来避免传统面相块的存储方式的overhead，使得上层应用可以直接访问PM。

### **一. NVM特性**
~~NVM是RAM而flash是ROM，~~<br>
NVM即非易失性存储器,所有在掉电后仍能保持其内容的内存组件都可以称为NVM,fast, byte-addressable, non-volatile<br>
NVM的读写速度比flash快，NVM是byte-addressable的而flash不是

### **二. consistency**
####保证一致性的三种方式：
1. copy-on-write: 先复制一个数据副本,在副本上进行修改,最后将指针指向新数据
2. journaling(logging): 通过写日志记录操作,根据日志进行恢复,需要执行写日志和写文件两次操作
3. log-structured updates: 以日志的形式组织文件系统并执行更新

####journaling的两种方式：
1. redo: 新数据被写入日志中并持久化，直到事务成功提交才写入文件系统中。
2. undo: 旧数据被写入日志中并持久化，新数据直接写入文件系统中，若事务失败则通过重写日志中的旧数据来回滚。
3. 比较：undo实现简单,对事务中的每个log entry都需要一次pm_wbarrier;redo对一次事务操作只需要两次pm_wbarrier,但对事务中的所有读操作都带来额外的overhead。PMFS采用undo。

####PMFS采用的方式：
对小的metadata的更新采用atomic in-place and fin-grained logging, 而对于文件数据的更新则采用CoW。
<br>
hybrid approach: atomic in-place updates and fine-grained logging for the (usually small) metadata updates, and CoW for file data updates。
<br>
atomic in-place updates:in order to minimize journaling overhead[8/16/64-bytes]

### **三. 三条指令**
1. clflush: 将cache中修改后的数据写进memory system中(但不一定能写进nvm）flush the cacheline
2. sfence: 保证存储操作的完成 ensure the completion of store
3. pm_wbarrier: 保证数据一定能写进nvm中。 ensure the durability of every store to PM
<br>
引入pm_wbarrier是为了解决clflush不能保证一定将数据写入nvm的问题，三条指令共同保证durability

### **四. 补充**
1. Mmap: Mmap in PMFS maps file data directly into the application’s virtual address space,出于性能的考虑。
2. 解决reorder问题：将log entry的大小设置成与一个cacheline相同(64bytes)，并且保证对同一个cacheline的写操作不会被reordered。
3. write protection: a). 利用SMAP,Prohibit writes into user area; b). PMFS write windows: mount as read-only, when writing CR0.WP is set to zero 

### **五. 课前习题**

---
> Why do memory reordering and cache play an important role in PMFS consistency? How does PMFS maintain consistency?

因为在保证metadata一致性时采用的方式是journalling，在标明记录的log entry durable的时候作为标志的gen_id必须最后写，因此需要避免编译器的memory reordering，否则可能造成gen_id标明为valid时log entry却不是的错误情况。cache保证了对同一个cacheline的写操作的顺序不被重排。

PMFS采用保证一致性的方式是对metadata使用logging，而对data使用Cow。
 
---

> There are three techniques to support consistency: copy-on-write, journaling and log-structured updates. Please explain the differences among them. Which technique does PMFS choose?

1. cow: 先复制一个数据副本,在副本上进行修改,最后将指针指向新数据
2. journaling: 通过写日志记录操作,根据日志进行恢复
3. log-structrued updates: 以日志的形式组织文件系统并执行更新
<br/>
cow即使在做很小的更新的情况下可能也需要先复制一个很大的数据副本造成额外的overhead；journaling需要执行两次写操作，一次写日志，一次写文件；log-structrued fs在log的末尾追加数据(顺序写)，写操作高效但随机读的效率低。
<br>
PMFS采用的方式是对小的metadata的更新采用atomic in-place and fin-grained logging, 而对于文件数据的更新则采用CoW。

---
> What granularity does PMFS use for log? Please explain the reason.

PMFS中log entry的大小和采用和一个cacheline一样的64 byte

因为利用两次pm_wbarrier或者checksum的方式来保证PMFS－Log有效的方式代价太大，作者将log entry的大小设置成与一个cacheline相同，并且利用cacheline中保证写顺序不被reorder的特性来保证log的validate。

---