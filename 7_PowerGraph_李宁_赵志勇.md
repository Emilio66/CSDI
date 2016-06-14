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
 
#### random vs greedy
* random placement   优点：可并行、分布式地执行，高效；  缺点：replica数量较多，通信开销大
* greedy placement  replica数量少，需要协调各程序，复杂，ingress（构建划分）时间较长

  #####greedy placement的两种实现方式:  
  * coordinated greedy placement： 少replication factor, 高runtime  
  * oblivious greedy placement： 优点兼而有之，独立运行，无额外通信开销，相对较少的replica，较少的运行时间
 
##课后题解答

