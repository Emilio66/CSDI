#PowerGraph: Distributed Graph-Parallel Computation on Natural Graphs

##助教提供的考点
1. 图计算遇到的问题（可结合PPT)
2. GAS模型
3. greedy vs random 划分比较
4. **课后题(重要)**

##概要（包含考点的解答）
图编码了关系，边和节点信息可以用来存储现实世界中的各种关系。     


####问题： 
自然图highly skewed，有power-law degree distribution现象（少数节点拥有极大多数边，大多数节点只有少量边），此外目前的按边划分的质量差导致
*   work imbalance(gather、scatter与degree数量成正比), 
*   partition(直接hash，随机划分，poor locality，高度数节点与低度数节点被同样地划分，不合理)，
*   通信开销不平衡（度数高的节点所在机器需要较多的通信）；
*   存储不均衡，一个节点的所有邻边信息可能超过机器容量；
*   computation（之前的计算模式是多个节点之间可以并行，单个节点内无法并行，对于高度数节点来说可扩展性不强）
   
图并行化抽象流行的两种方式：
--使用消息 Pregel
--共享状态 GraphLab
Pregel向单个worker发送大量消息给邻居节点，成本较高。同步执行但容易产生straggler，straggler可以理解为执行比较慢的节点。
GraphLab共享状态时异步执行，需要大量锁。会触到图的大部分，并且对于单台机器边的元数据太大
-->
GraphLab和Pregel是不适合处理这种natural graphs 
主要的两大挑战是高纬度的点和低质量的分区策略。

PowerGraph的特点
第一，提出了GAS计算模型，将高维度的点进行并行化
第二是采用点切分策略，来保证整个集群的均衡性，该策略对大量密率图分区是非常高效的。

#### GAS模型 
这是paper提出的一种图计算程序的三个概念上的阶段：
*   gather，收集计算所需的邻接节点和边的信息
*   apply，将结果作用于中心节点
*   scatter，分发新的中心节点的值给各条邻边
 

#### 划分方式
*   edge-cut 把边切开，节点平均分配，有的点有很多边，imbalance  

 例子：pregel（synchronous bulk message passing）  graphLab asynchronous shared memory
*   vertex-cut 边平均分配，点可以跨机器，有mirror节点    

 例子：powergraph （balanced p-way vertex-cut)


PowerGraph使用的不是边切分，边切分前面已经提到会同步大量的边的信息。而是采用点切分，点切分只要同步一个点的节点信息。
 
#### random vs greedy
* random placement   优点：可并行、分布式地执行，高效；  缺点：replica数量较多，通信开销大
* greedy placement  replica数量少，需要协调各程序，复杂，ingress（构建划分）时间较长、最小化机器所跨的机器数目

  #####greedy placement的两种实现方式:  
  * coordinated greedy placement：速度慢质量高 少replication factor, 高runtime  （维护一张全局u顶点放置的历史纪录表，在执行贪心切分之前都要去查询这张表，在执行的过程中需要更新这张表）
  * oblivious greedy placement： 速度快质量稍低 优点兼而有之，独立运行，无额外通信开销，相对较少的replica，较少的运行时间（每台机器自己维护这张表，不需要做机器间的通信）
 
总结：
协同的贪婪分区：机器跨度最小，但构建时间最长。
随机策略：构建时间短，但平局的机器跨度最大。
Oblivious的贪婪分区策略：折中 
##课后题解答


1.How skewed degree distribution challenges the original graph parallel computation? Give a brief summary of the computation procedure and then analysis the challenges.

PowerGraph提出了自己的一套计算模型，叫GAS分解。G是Gather的意思，A是Apply的意思，S是Scatter的意思。
GAS分解过程如下，
Gather：收集邻居信息 先收集同一台机器的信息，然后对不同主机收集的信息进行汇总。得到最后的求和信息。
Apply：对中心点应用收集点的值，得到y一撇
Scatter（分散）：更新邻居点和边，触发邻居点进行下一轮迭代。 

自然图并行化计算的挑战：
work balance：图的 skewed degree distribution 会导致工作分布不平衡
Partitioning：skewed degree distribution导致很难分区
Communication：skewed degree distribution导致信息交流不对称
Storage：skewed degree distribution导致某个点的度数超过机器的存储量
Computation：由于限制度数高的节点的可伸缩性，现有的图计算不能并行化计算个人的vertex-programs

2.In your opinion, what are the advantages of graph parallel computation comparing with traditional data parallel processing (e.g. map-reduce)?
图计算相对于传统数据并行处理的优势：
在并行计算中，图计算提供非常灵活的对于数据之间管子的抽象描述，能够获得数据间更深层次的关系。
随着数据集的增长，复杂的数据计算模型和存储已经达到单个机器的极限，图计算可以提高并行性且降低网络通信和存储成本
（感觉还有优势，欢迎补充）

3.Brief explain vertex-cut and G A S steps using following small graph (each src-dst pair is a directed edge of the graph). Assume we have 3 nodes, and hash(vertex)=vertex%3, hash(src,dst)= (src+dst)%3. You may need to draw a graph.
0-1   0-2   0-3 
0-4   1-3   1-4 
2-4   2-5   3-5

使用点切割将6个vertices划分成3个nodes（0,1,2），并使用hash(src,dst)= (src+dst)%3将边划分到3个nodes
![alt text](/7-3-1.png "graph1")
GAS steps:
选择vertex 0 in node 0 作为master（主节点）, 其余mirrors（备份节点）.
1.Gather：
	gather在每个机器上单独运行，即从邻近的vertices收集信息（sum过程），将accumulator从mirror传输到master 
2.Apply:
	master更新vertices的数据，然后将更新后的数据传输到各个mirrors
3.Scatter:
	将更新后的数据发送到邻近的vertices，并进行下一次迭代。
![alt text](/7-3-2.png "result")