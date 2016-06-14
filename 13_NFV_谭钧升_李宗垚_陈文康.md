#The Click Modular Router

##助教提供的考点
1. 为什么要提出这个技术？与传统的区别？
	Unfortunately, most routers have closed, static, and inflexible designs. 现有的大多数路由器的设计都是封闭的，静态的和刻板的。
    Network administrators may be able to turn router functions on or off, but they cannot easily specify or even identify the interactions of different functions.
    网络管理员很难指定甚至定义不同functions之间的交互。
    Furthermore, network administrators and third  party software vendors cannot easily implement new  functions.
    而且，要实现新的functions很难。
    与传统的区别：（1）灵活:很容易添加新的功能 （2），模块化：通过组合元素来实现功能； （3）， Open：允许开发者添加元素（Elements);（4），效率：与硬件性能相差不大。

2. Pull和Push
	push和pull是Click里elements直接的两种连接方式。
	Push是从源Element到下游的元素，通过事件触发，比如有包到达。在Push连接方式里，上游的元素会提交一个包到下游的元素；
    Pull是从目的地元素到上游元素。在Pull的连接方式里，是下游的元素发送包请求到上游的元素。当输出接口准备好发包时，会开始一个pull过程：该过程会一直向后遍历图直到遇到某个元素‘吐出’一个包。

3. 包调度
	在Click里，调度器就是一个有着多个输入端口，一个输出端口的pull element，
	并通过一定的调度策略来决定从这多个input进来的包应该如何共享这个Output。
	在论文里提到click实现了两个调度器：Round-Robin Sched 和 PrioSched (Priority Scheduler)。 Round-Robin Sched 就是对input进行轮询。 PrioSched (Priority Scheduler)就是每次都从第一个input开始pull packets。
	
4. Dropping Policies
	Click通过队列元素来实现一个简单的Dropping策略， 也就是当包数目超过配置的最大长度时，这些包都会被扔掉。
	论文提到的Dropping策略：（1），RED：Random Early Detection。 该Element以下游的最近的队列长度作为Dropping的依据。(2), RED over multiple queues: 如果RED的下游有多个队列，那就将这些队列的长度都加起来作为Dropping的依据；（3），weight RED: 每个包根据它的优先级有不同的Drop的概率。
	
5. NFV:Network Function Virtualization
   Network Functions(Middleboxes): Firewall, IDS, DNS。网络功能虚拟化就是虚拟化这些网络功能，也就是通过软件模拟实现这些网络功能。
   或者说是：运行在云设施的网络服务。
    NFV目标：（1）省钱：使用更便宜的商业服务器来实现网络功能；减少专有硬件的使用，从而降低能耗，并且能够方便地维护。（2）赚钱：加速网络服务部署；
    网络基础设施作为服务；


##提纲

##课后题解答

