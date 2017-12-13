## ReentrantLock and AQS实现原理

提到JAVA加锁，我们通常会想到synchronized关键字或者是Java Concurrent Util（后面简称JCU）包下面的Lock，今天就来扒一扒Lock是如何实现的，比如我们可以先提出一些问题：当我们通实例化一个ReentrantLock并且调用它的lock或unlock的时候，这其中发生了什么？如果多个线程同时对同一个锁实例进行lock或unlcok操作，这其中又发生了什么？

### 什么是可重入锁？

ReentrantLock是可重入锁，什么是可重入锁呢？可重入锁就是当前持有该锁的线程能够多次获取该锁，无需等待。可重入锁是如何实现的呢？这要从ReentrantLock的一个内部类Sync的父类说起，Sync的父类是AbstractQueuedSynchronizer（后面简称AQS）。

### 什么是AQS？

AQS是JDK1.5提供的一个基于FIFO等待队列实现的一个用于实现同步器的基础框架，这个基础框架的重要性可以这么说，JCU包里面几乎所有的有关锁、多线程并发以及线程同步器等重要组件的实现都是基于AQS这个框架。AQS的核心思想是基于volatile int state这样的一个属性同时配合Unsafe工具对其原子性的操作来实现对当前锁的状态进行修改。当state的值为0的时候，标识改Lock不被任何线程所占有。

### ReentrantLock锁的架构

ReentrantLoc的架构相对简单，主要包括一个Sync的内部抽象类以及Sync抽象类的两个实现类。上面已经说过了Sync继承自AQS，他们的结构示意图如下：



上图除了AQS之外，我把AQS的父类AbstractOwnableSynchronizer（后面简称AOS）也画了进来，可以稍微提一下，AOS主要提供一个exclusiveOwnerThread属性，用于关联当前持有该锁的线程。另外、Sync的两个实现类分别是NonfairSync和FairSync,由名字大概可以猜到，一个是用于实现公平锁、一个是用于实现非公平锁。那么Sync为什么要被设计成内部类呢？我们可以看看AQS主要提供了哪些protect的方法用于修改state的状态，我们发现Sync被设计成为安全的外部不可访问的内部类。ReentrantLock中所有涉及对AQS的访问都要经过Sync，其实，Sync被设计成为内部类主要是为了安全性考虑，这也是作者在AQS的comments上强调的一点。

### AQS的等待队列

作为AQS的核心实现的一部分，举个例子来描述一下这个队列长什么样子，我们假设目前有三个线程Thread1、Thread2、Thread3同时去竞争锁，如果结果是Thread1获取了锁，Thread2和Thread3进入了等待队列，那么他们的样子如下：



AQS的等待队列基于一个双向链表实现的，HEAD节点不关联线程，后面两个节点分别关联Thread2和Thread3，他们将会按照先后顺序被串联在这个队列上。这个时候如果后面再有线程进来的话将会被当做队列的TAIL。

### 1）入队列

我们来看看，当这三个线程同时去竞争锁的时候发生了什么？
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
解读：

三个线程同时进来，他们会首先会通过CAS去修改state的状态，如果修改成功，那么竞争成功，因此这个时候三个线程只有一个CAS成功，其他两个线程失败，也就是tryAcquire返回false。

接下来，addWaiter会把将当前线程关联的EXCLUSIVE类型的节点入队列：
```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
解读：

如果队尾节点不为null，则说明队列中已经有线程在等待了，那么直接入队尾。对于我们举的例子，这边的逻辑应该是走enq，也就是开始队尾是null，其实这个时候整个队列都是null的。
```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
解读：

如果Thread2和Thread3同时进入了enq，同时t==null，则进行CAS操作对队列进行初始化，这个时候只有一个线程能够成功，然后他们继续进入循环，第二次都进入了else代码块，这个时候又要进行CAS操作，将自己放在队尾，因此这个时候又是只有一个线程成功，我们假设是Thread2成功，哈哈，Thread2开心的返回了，Thread3失落的再进行下一次的循环，最终入队列成功，返回自己。

### 2）并发问题

基于上面两段代码，他们是如何实现不进行加锁，当有多个线程，或者说很多很多的线程同时执行的时候，怎么能保证最终他们都能够乖乖的入队列而不会出现并发问题的呢？这也是这部分代码的经典之处，多线程竞争，热点、单点在队列尾部，多个线程都通过【CAS+死循环】这个free-lock黄金搭档来对队列进行修改，每次能够保证只有一个成功，如果失败下次重试，如果是N个线程，那么每个线程最多loop N次，最终都能够成功。

### 3）挂起等待线程

上面只是addWaiter的实现部分，那么节点入队列之后会继续发生什么呢？那就要看看acquireQueued是怎么实现的了，为保证文章整洁，代码我就不贴了，同志们自行查阅，我们还是以上面的例子来看看，Thread2和Thread3已经被放入队列了，进入acquireQueued之后：

对于Thread2来说，它的prev指向HEAD，因此会首先再尝试获取锁一次，如果失败，则会将HEAD的waitStatus值为SIGNAL，下次循环的时候再去尝试获取锁，如果还是失败，且这个时候prev节点的waitStatus已经是SIGNAL，则这个时候线程会被通过LockSupport挂起。

对于Thread3来说，它的prev指向Thread2，因此直接看看Thread2对应的节点的waitStatus是否为SIGNAL，如果不是则将它设置为SIGNAL，再给自己一次去看看自己有没有资格获取锁，如果Thread2还是挡在前面，且它的waitStatus是SIGNAL，则将自己挂起。

如果Thread1死死的握住锁不放，那么Thread2和Thread3现在的状态就是挂起状态啦，而且HEAD，以及Thread的waitStatus都是SIGNAL，尽管他们在整个过程中曾经数次去尝试获取锁，但是都失败了，失败了不能死循环呀，所以就被挂起了。当前状态如下：



锁释放-等待线程唤起

我们来看看当Thread1这个时候终于做完了事情，调用了unlock准备释放锁，这个时候发生了什么。
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
解读：

首先，Thread1会修改AQS的state状态，加入之前是1，则变为0，注意这个时候对于非公平锁来说是个很好的插入机会，举个例子，如果锁是公平锁，这个时候来了Thread4，那么这个锁将会被Thread4抢去。。。

我们继续走常规路线来分析，当Thread1修改完状态了，判断队列是否为null，以及队头的waitStatus是否为0，如果waitStatus为0，说明队列无等待线程，按照我们的例子来说，队头的waitStatus为SIGNAL=-1，因此这个时候要通知队列的等待线程，可以来拿锁啦，这也是unparkSuccessor做的事情，unparkSuccessor主要做三件事情：

将队头的waitStatus设置为0.

通过从队列尾部向队列头部移动，找到最后一个waitStatus<=0的那个节点，也就是离队头最近的没有被cancelled的那个节点，队头这个时候指向这个节点。

将这个节点唤醒，其实这个时候Thread1已经出队列了。

还记得线程在哪里挂起的么，上面说过了，在acquireQueued里面，我没有贴代码，自己去看哦。这里我们也大概能理解AQS的这个队列为什么叫FIFO队列了，因此每次唤醒仅仅唤醒队头等待线程，让队头等待线程先出。

### 羊群效应

这里说一下羊群效应，当有多个线程去竞争同一个锁的时候，假设锁被某个线程占用，那么如果有成千上万个线程在等待锁，有一种做法是同时唤醒这成千上万个线程去去竞争锁，这个时候就发生了羊群效应，海量的竞争必然造成资源的剧增和浪费，因此终究只能有一个线程竞争成功，其他线程还是要老老实实的回去等待。AQS的FIFO的等待队列给解决在锁竞争方面的羊群效应问题提供了一个思路：保持一个FIFO队列，队列每个节点只关心其前一个节点的状态，线程唤醒也只唤醒队头等待线程。其实这个思路已经被应用到了分布式锁的实践中，见：Zookeeper分布式锁的改进实现方案。

### 总结

这篇文章粗略的介绍一下ReentrantLock以及锁实现基础框架AQS的实现原理，大致上通过举了个三个线程竞争锁的例子，从lock、unlock过程发生了什么这个问题，深入了解AQS基于状态的标识以及FIFO等待队列方面的工作原理，最后扩展介绍了一下羊群效应问题，博主才疏学浅，还请多多指教。

### Support or Contact

Having trouble with Pages?

lfl969605@126.com
