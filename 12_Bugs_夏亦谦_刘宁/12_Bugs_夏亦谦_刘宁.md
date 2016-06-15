#Bugs, Towards Optimization-Safe Systems: Analyzing the Impact of Undefined Behavior 

##助教提供的考点
1. 主要看懂上课的slides
2. 什么是Undefined behavior？
3. 什么是Unstable code？
4. 如何检测Unstable code？

##概要
Undefined behaviour: 是一些程序错误，如空指针解引用，缓冲区溢出、use after free等。  
Unstable code: 由于被定义为是undefined behavior，从而被编译器略过的代码。

类似于这种undefined behavior在C编译器中还有很多：
* Pointer overflow:   if ( p + 100 < p)
* Signed integer overflow:    if (x + 100 < x) 
* Oversized shift:    if (!(1 << x))
* Null pointer dereference:    *p;  if (p)
* Absolute value overflow:    if (abs(x) < 0)

![alt text](/12_Bugs_夏亦谦_刘宁/example.png)  

上面的代码中，buf + off < buf 就是一个undefined behavior：当off非常大的时候，会导致溢出，但gcc会认为(buf + off)总是小于buf_end的。所以当溢出的时候，gcc检测不到，会绕过这一段溢出检测代码，然后去读取buf段以外的内存。

本文对12种C/C++编译器做了测试，发现：
1. 有一些编译器会悄悄地把unstable code给移除掉；
2. 不同的编译器会有不同的编译规则，不同版本的编译器对同一段代码会有不同的处理方式。
因此，需要一个成体系的approach来解决以上问题。

###本文的解决方案-方法论
令Assumption Δ = 代码中不会出现undefined behavior  
用公式表示Assumption Δ: 
![alt text](/12_Bugs_夏亦谦_刘宁/equation.png)  
Reach(e): when to reach and execute code fragment e  
Undef(e): when to trigger undefined behavior at e  

STACK会在Assumption Δ被允许和不允许的情况下分别模拟编译。  
Step1 - 先模拟假设不成立的情况进行一次编译；  
Step2 - 模拟假设成立的情况进行一次编译；  
Step3 - 查看前两步的执行结果有没有区别，有区别的地方就是unstable code。

###举个栗子
![alt text](/12_Bugs_夏亦谦_刘宁/example2.png)  
在这个代码段中，当((y==0) or (x == -1 or y == INT_MIN))时，x/y可能出现溢出。
因此，Assumption Δ：(y != 0) 且 (y != -1 or x != INT_MIN)

根据 Δ = ∀e:Reach(e) → ¬Undef(e)，可以对以上的三行代码列出公式如下，最终得到Δ的具体表达式。
![alt text](/12_Bugs_夏亦谦_刘宁/example3.png)  
解释说明一下Δ怎么计算出来的：Δ = ∀e:Reach(e) → ¬Undef(e)的意思是：对于任意的代码行e，在执行到了e的情况下，不会触发undefined behavior。由于本例中的代码段一共有三行，所以是对三行取交集。第一行：Reach(e)为true，Undef(e)为(y == 0) or (x == -1 and y == INT_MIN)。第二行：Reach(e)为true，由于本行不存在undefined behavior，因此Undef(e)为false。第三行：由于本行是在第二行成立的条件下才会执行到，因此Reach(e)=(y == -1 && x < 0 && x/y < 0)，同时本行也没有undefined behavior，因此Undef(e)为false。

###本文的Limitation
1. 如果执行第二步时得不到准确的结果，那么会漏报一些unstable code；
2. 如果执行第一步时得不到准确的结果，就会产生误报(false warning / false positive)。

###如何避免Unstable Code
* 对于程序员来说，通过fix bug或者去掉一些会被编译器当做是undefined behavior的代码；
* 对于编译器来说，可以集成一些现有的bug-finding的工具，或者利用STACK的方式来判定unstable code。

















