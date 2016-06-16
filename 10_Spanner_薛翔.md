# Spanner: Google’s Globally-Distributed Database

## 考点
1. SQL, noSQL, newSQL演化过程
2. Spanner技术细节
3. 课后题

## SQL -> noSQL -> newSQL 
    [为什么](http://dataconomy.com/sql-vs-nosql-vs-newsql-finding-the-right-solution/)
    SQL：使用广泛，保证ACID；扩展性差，过于通用，调试复杂;
    noSQL：最终一致性，扩展性好，动态调整schema；代价是ACID的弱化;
    newSQL：强一致性，事务支持，SQL语义和工具，性能好；通用性还是没SQL好

## Spanner
### **1. Overview**
#### 现状
    不适用于BigTable的应用：“complex, evolving schemas”，或要求所有副本保持强一致性。
    Megastore：poor write throughput
#### 区别于BigTable
- 从简单的key-value store加强到temporal multi-version database；
- 数据以半关系型的table组织；
- 支持txn语义；
- 支持SQL查询。

#### 两个特性
- 其上的应用程序能够动态配置replication，达到负载均衡和降低延迟；
- External Consistency
    定义：分布式事务系统中，txn1的commit先于txn2的start，那么txn1的提交时间应小于txn2的提交时间。即能看见txn2的时候一定要能看见txn1。

### **2. Implementation**
#### 系统架构
![](/img/10-1.png "spannerServerOrganization")
- universe：Spanner的整个部署；
- zone：等同于其下BigTable的部署节点，是管理配置的单位，也是物理隔离的单位；
- zonemaster：每个zone都有一个，负责将数据分配给当前zone的spanserver；
- spanserver：每个zone有成百上千个，负责为client提供数据服务；
- universe master：单例，维护所有zones；
- placement driver：单例，负责在zones之间迁移数据。

#### Spanserver架构
![](/img/10-2.png "spannerSoftwareStack")
- 每个tablet上维护一个Paxos状态机
- 写请求由Paxos选出的leader负责；读请求由任一足够up-to-date的replica执行都行
- leader持有lock table来执行2PL提交
- txn mngr负责处理跨Paxos group的txn（2PL）

#### Directories
    dir是数据放置的单元，其下所有数据有一致的备份设置

#### Data Model
    基于directory-bucketed key-value mappings，主键作为key，其他作为value

### **3. TrueTime**
#### API
    TT.now()：返回一个时间段[earliest, latest]，保证被调用的一刻的实际时间处在这个范围内
    TT.after(t), TT.before(t)：检查时间t是否已经成为“过去”或仍处在“未来”，即是否小于earliest或大于latest
#### 实现方式
    使用GPS和原子钟来保证TT.now()准确性
    *GPS互相同步但易受干扰：原子钟相对稳定但一段时间不同步会导致TT.now()时间段变大（原子钟的频率会有微小差异）

### **4. Concurrency Control**

#### 主要的txn类别
- read-write txn: 普通的读写txn；
- read-only txn: 确定只读的txn。不拿锁，不block接下来的read-write txn，选择足够up-to-date的replica执行都行。
- snapshot reads: 读历史数据的txn。不拿锁，选择足够up-to-date的replica执行都行。

#### ﻿Read-Write Txns
1. （client执行部分）拿锁；
2. 执行read&write；
3. 开始2PC，选择coordinator group，将修改发送给coordinator leader；
4. （所有non-coordinator-participant leader执行部分）选择大于本地最近一次提交的timestamp作为prepare timestamp返回给coordinator leader；
5. （coordinator leader执行部分）获得相应的写锁；
6. 获取所有participant leader的prepare timestamps，选择不小于所有prepare timestamps的s作为commit timestamp，此s还应大于TT.now().latest和本地最近txn的commit timestamp；
7. 等待s < TT.now().earliest，即TT.after(s)，确保所有在s之前的txn都全局生效；
8. 以s为commit timestamp提交当前txn，并反馈client；
9. 释放锁。

#### Read-Only Txns
    首先，提取所有会被读到的key作为scope，然后分类讨论：
    *以下两种处理都能保证这次读在所有已全局生效的写之后
1. 如果scope都落在一个Paxos group：将这个RO txn发送给group leader；leader调用LastTS()获取最近的write timestamp作为RO txn的timestamp并执行
2. 如果scope跨多个Paxos groups：读取TT.now().latest作为当前RO txn的timestamp并执行
    

#### Schema-Change Txns
    通过TT，选取未来的timestamp作为该txn提交时间，记为s；
    所有在s之前的txn正常执行；在s之后的被blocked，直到TT.after(s)==true再执行

## 课后题
1. What is external consistency? What’s the difference between external consistency and serializability?
2. How does Spanner achieve the external consistency?
3. What will happen if the TrueTime assumption is violated? How the authors argue that TrueTime assumption should be correc