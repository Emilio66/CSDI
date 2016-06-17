## 提出的新技术、为什么要提出、解决什么问题、优缺点是什么、challenge、技术细节
这篇文章就是提出了non-scalable lock是不好的，提出要用scalable lock提到前者，没有新技术，之前两者都是有的。
原因是non-scalable lock哪怕在N个核的时候表现不错，也很有可能在N+1或者N+2个核的时候突然collapse。
scalable lock解决的就是这个问题，优点就是不会产生这种突然的大幅度的collapse，锁的性能不会随着核的数量增加而下降太大。缺点就是要对现有的源代码做改动，但是改动并不复杂。
技术细节见第二个课后题。

## 助教提供的复习点：为什么non-scalable lock是dangerous的？性能为什么会突然下降?
dangerous在上面已经说过了，为什么性能突然下降，见第一个课后题。

## 课后习题
1. Why does the performance of ticket lock collapse with a small number of cores? (Hint: cache coherence protocol)
这是因为在ticket lock中，会需要记录两个变量，一个是now_serving表示正在使用lock的ticket（一个整数），另一个是next_ticket记录着当前最后一张ticket（就是拿票等待的核的号码）。当任何一个核去拿锁的时候，都会需要读取now_serving这个变量判断跟自己的ticket是否相等。这样一来，每个核都会对now_serving做cache，一旦这个锁被释放，ticket lock中的now_serving就会增加1，这个操作会invalidate所有核的cache里的now_serving，这会触发所有的核来重新读取now_serving这个值所在的cacheline，论文说明了在现有的架构中，这个read会被串行化处理，一个一个来，这就导致消耗的时间与等待锁的核的数量呈线性增长关系。
至于为什么会突然发生collapse，大家可以参看3.4的第一个implication。我觉得是说，当很多核想得到同一个锁，并且进入了contend状态之后，一个锁从一个核转移到另一个核的时间是与等待锁的核的数量呈线性关系的，但是这个时间会极大地增加串行部分代码的长度(the length of the serial section),所以当某个锁积累到一定量的waiter，就会突然collapse。

2. To mitigate the performance collapse problem of ticket lock, we can replace it with MCS lock. Please describe the MSC lock algorithm in C.

mcs_node{
      mcs_node next;
      int is_locked;
}

mcs_lock{ 
      mcs_node queue;
}

function Lock(mcs_lock lock, mcs_node my_node){
      my_node.next = NULL;
      mcs_node predecessor = fetch_and_store(lock.queue, my_node);
      //if its null, we now own the lock and can leave, else....
      if (predecessor != NULL){
          my_node.is_locked = true;
          //when the predecessor unlocks, it will give us the lock
          predecessor.next = my_node; 
          while(my_node.is_locked){}
}

function UnLock(mcs_lock lock, mcs_node my_node){
      //is there anyone to give the lock to?
      if (my_node.next == NULL){
            //need to make sure there wasn't a race here
            if (compare_and_swap(lock.queue, my_node, NULL)){
                 return;
            }
            else{
                 //someone has executed fetch_and_store but not set our next field
                 while(my_node.next==NULL){}
           } 
      }
     //if we made it here, there is someone waiting, so lets wake them up
     my_node.next.is_locked=false;
}

3. Why does MCS lock have better scalability than ticket lock?
从上面的代码其实可以很快地找到答案，在ticket lock中是所有的核都去读一个变量来判断自己是不是能够拿到锁，但是MCS不用，MCS是当一个正在使用锁的核把锁放掉之后，它会主动检查是不是有核在等这把锁，如果有会去通知这个核，代码上就是前一个核会去修改后一个核的is_locked变量，后一个核会一直spin在这个变量上，当这个变量被修改，它就知道自己能够拿到这把锁了。
所以，每个核都spin在自己的is_locked变量上，而不是全局去读某一个变量，这样就会有更好的scalability。
