### ReentrantLock源码剖析

​		这里又是看了忘忘了看系列之ReetrantLock，今天趁着有时间记录下ReentrantLock源码的学习过程。这篇博客主要记录以下几个方面内容。**欢迎各位多提建议或者意见**

​		1、ReetrantLock和Sync的继承结构

​		2、ReetrantLock构造函数们及AQS的核心属性

​		3、ReetrantLock锁的使用示例

​		4、ReetrantLock的**原理**及**核心**方法和设计思想

##### 1、ReetrantLock和Sync的继承结构

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ReetrantLock/images/ReentrantLock.png)

​		上图就是ReetrantLock和Sync的继承结构及关系，ReetrantLock内部有一个成员变量Sync，Sync又继承自AQS（AbstractQueuedSynchronizer）。ReetrantLock内部同时也有两个内部类FairSync和NonfairSync（都是Sync的子类），两个类内部只有两个方法，都是重写了父类的方法，分别是lock()和tryAcquire()。

##### 2、ReetrantLock构造函数们及AQS的核心属性

###### 2.1 先介简单的绍构造函数

```java
// 常用构造函数，内部创建非公平锁
public ReentrantLock() {
        sync = new NonfairSync();
}

// 通过手工指定是公平锁还是非公平锁，true为公平锁，false为非公平锁
public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
}
```

###### 2.2 介绍ReetrantLock的属性及相关属

* ReentrantLock自身属性

```java
/** Synchronizer providing all implementation mechanics */
private final Sync sync;
```

* AQS的核心属性

  ​		在介绍AQS的属性之前先介绍下AQS的结构和用处，AQS是许多同步器的核心，内部大量使用了CAS机制。AQS是实现大部分同步需求的基础。

  ​		同步器的主要使用方式是继承，子类通过继承并实现他的抽象方法来管理同步状态。ReentrantLock、ReentrantReadWriteLock和CountDownLatch等同步器组件的核心都是AQS。

  ​		AQS使用队列来管理等待锁的线程，内部类似一个FIFO队列，每一个节点使用Node来表示。结构如下图所示。

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ReetrantLock/images/AQS.png)

```java
// AQS下属性为同步器的核心属性
private transient volatile Node head;

private transient volatile Node tail;

private volatile int state;

// Node结构中的属性，本次只分析和ReetrantLock相关的属性
static final class Node {
        // 排他锁模式
        static final Node EXCLUSIVE = null;

        // 取消状态
        static final int CANCELLED =  1;
        // 此状态表示其还有后续节点
        static final int SIGNAL    = -1;
      	// 等待状态，会被赋值为上面几种状态之一
        volatile int waitStatus;
        // 前置节点
        volatile Node prev;
	    // 后置节点
        volatile Node next;
	    // 节点所依附的线程
        volatile Thread thread;

}
```

##### 3、ReetrantLock锁的使用示例

​		下面我来演示一个我们常用的示例，随后通过这个示例来展开分析核心原理。

```java
public class Test{
    public static void main(String [] args){
        // 创建锁
        ReentrantLock myLock = new ReentrantLock();
        // 在需要的地方加锁
        myLock.lock();
        try {
            // 在加锁后做一些感兴趣的事情
            System.out.println("fuhang do something");
            throw new RuntimeException("Oh,that's too bad !");
        }catch (Exception e){
            // do something
        }finally {
            // 最后一定要手动释放锁
            lock.unlock();
        }
    }
}
```

​		上面就是ReetrantLock的一个简单示例，这里插播记录一个ReetrantLock对比Synchronized关键字的优势，比如A线程要获取B锁，获取到B锁后需要再获取C锁，获取到C锁后需要释放B锁这种场景下，使用ReentrantLock就很好控制，而Synchronized相对来说就不是这么方便了。

##### 4、ReetrantLock的**原理**及**核心**方法和设计思想

​		下面我们剖析使用示例中的方法一步一步来学习ReetrantLock的原理。

###### 4.1 加锁过程

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ReetrantLock/images/addLock.png)

​		① 先调用ReetrantLock.lock()方法，这个方法内部调用了属性变量sync的lock方法

```java
// ReetrantLock 内部调用了Sync的子类的lock方法，在这里就是NonfairSync类的lock方法
public void lock() {
     sync.lock();
}
```

​		②调用Sync类的子类NonfairSync的lock()方法。

​	 下面这个方法第一个if条件尝试获取锁，在这里获取锁就是使用 CAS 机制将 AQS 类中的 state 属性变量从 0 变为 1 （ReentrantLock中的同步器（Sync）的state属性为0 时候表示无锁，大于 0 时候表示加锁状态），如果修改成功则表示获取到锁，然后调用`AbstractOwnableSynchronizer.setExclusiveOwnerThread(Thread owner)`方法设置获取到锁的线程。

​	 如果获取锁失败则进入else代码，执行acquire(1)。

```java
// NonfairSync.lock()方法
final void lock() {
       if (compareAndSetState(0, 1))
           setExclusiveOwnerThread(Thread.currentThread());
       else
           acquire(1);
}
```

​		③ 调用 `acquire(1)`方法，在这个方法里面直接进入if判断，在if语句里面先执行`tryAcquire(arg)`方法尝试再次获取锁和处理重入锁逻辑，若失败则再执行`addWaiter()`和`acquireQueued(Node,int)`方法。若两个条件都为真的情况下执行自我中断方法`selfInterrupt()`中断当前线程。

```java
public final void acquire(int arg) {
      if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
}
```

​		④ 调用`NonfairSync.tryAcquire(int)`方法，此方法内部调用`nonfairTryAcquire(int)`方法。

```java
protected final boolean tryAcquire(int acquires) {
      return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    	    // 获取当前线程引用
            final Thread current = Thread.currentThread();
    	    // 获取锁的状态
            int c = getState();
    	    // 如果是 0 则说明是无锁状态，CAS修改锁状态尝试获取锁，成功返回true
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }else if (current == getExclusiveOwnerThread()) {
                // 这里是处理重入锁的逻辑，重入一次就将state+1，最后返回true
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
    	    // 既不是重入锁也不可获取到锁的情况下返回false 
            return false;
}
```

​		⑤ 调用addWaiter(Node)方法。方法的简介为将当前线程以给定模式加入队列，此处的Node参数用来表示模式，Node.EXCLUSIVE代表互斥锁，Node.SHARED代表共享锁。

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ReetrantLock/images/addNode.png)

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // 尝试快速加入队列，若失败则调用enq方法入队
        Node pred = tail; // 获取队尾元素
        if (pred != null) { // 如果队尾不为空（也就是说队列存在）
            node.prev = pred; // 设置节点的前置节点未队尾节点
            if (compareAndSetTail(pred, node)) { // CAS设置队尾元素为node，其他线程可能在做同样操作
                pred.next = node; // 将原队尾元素的next指向新的队尾元素node
                return node;
            }
        }
    	// 如果上面尾部不存在（队列没初始化）
    	// 或者CAS设置尾部时候失败（说明存在竞争，别的线程设置成功了）
    	// 则调用enq(Node)方法将节点加入队列
        enq(node);
        return node;
}
```

​		⑥ enq(Node) 方法，此方法用来初始化队列或者死循环到成功将节点加入队列

```java
private Node enq(final Node node) {
        for (;;) {
            // 获取尾部节点
            Node t = tail;
            // 如果尾部节点为空，则尝试初始化队列
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 下面逻辑和addWaiter()中快速入队逻辑相同
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
}
```

​		⑦ `acquireQueued(Node,int)`方法，此方法在尝试获取锁失败后并且addWaiter()方法成功将包装当前线程的Node节点加入队列后调用。这个方法是每个没拿到锁并且被加入到等待队列的线程最后进入的方法，在这个方法里面使用死循环来响应唤醒信号或者中断信号。

```java
final boolean acquireQueued(final Node node, int arg) {
    	// 锁获取是否失败
        boolean failed = true;
        try {
            // 中断标志
            boolean interrupted = false;
            // 死循环响应
            for (;;) {
                // 这里首先获取当前节点前面的节点，因为一般都是前面节点唤醒后面节点的
                final Node p = node.predecessor();
                // 如果前置节点是头结点且当前线程获取锁成功
                if (p == head && tryAcquire(arg)) {
                    // 将头结点设置为当前节点
                    setHead(node);
                    // 前置节点的下一个节点引用设为null，帮助垃圾回收器回收
                    p.next = null; // help GC
                    // 将失败标志置位false，表示获取锁成功
                    failed = false;
                    // 返回中断标志，交由后续处理
                    return interrupted;
                }
                // 获取锁失败后执行两个方法，这两个方法在下面分析
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                    interrupted = true;// 此标志返回给最上层做后续处理
            }
        } finally {
            // 如果执行到此失败标志为true，则要做此节点的后续处理工作
            if (failed)
                cancelAcquire(node);
        }
}

// 此方法将node节点设置为头节点，并把node自身的前置节点引用和线程引用置空
private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
}
```

​		⑧ `shouldParkAfterFailedAcquire()`方法本意是判断是否在获取锁失败后应该阻塞线程，参数为线程对应节点及前置节点。什么样的方法可以安全的被阻塞呢？ 

​		当一个节点的waitStatus为SIGNAL时候，就表明它自身还有后续节点，在释放锁的时候需要唤醒后续节点。那一个节点能安全被阻塞就需要前置节点自身知道它有后续节点。那么此方法主要确认前置节点的waitStatus是否为SIGNAL，若不是则将前置节点的waitStatus修改为SIGNAL并等待下次进入方法继续判断。

```java 
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    	// 获取前置节点的等待状态，默认情况下为 0
        int ws = pred.waitStatus;
    	// 如果前置节点状态为SIGNAL，那么说明本节点可以安全阻塞
    	// 此状态下前序节点释放锁时候会唤醒后续节点
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) { // 如果前序节点的状态大于 0 , 那么就是 cancelled 待取消状态
            
            // 那么尝试往前继续找状态<=0的前置节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            // 找到后将前置节点的下一个节点指向本节点
            pred.next = node;
        } else {
            // 走到这里说明前置节点正常，但是前置节点状态不为SIGNAL
            // 那么将前置节点的状态使用CAS设置为SIGNAL，让前序节点释放锁时候知道
            // 它后面还有在等待的节点需要通知
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
    	// 返回false 等待下一次进入此方法
        return false;
 }
```

​		⑨ `parkAndCheckInterrupt()`方法主要是为了阻塞线程和响应唤醒及中断信息。其中有个点需要讲解下:`Thread.interrupted()`方法是个静态方法。

​		`Thread.interrupted()`内部调用了`currentThread().isInterrupted(true)`方法，此方法判断当前线程线程的中断标志是否为true，若为true，返回当前线程中断标志并将当前线程标志设置为false，若为false，返回当前线程中断标志后不做任何操作。若用户线程只调用一次`thread.interrupt()`方法，那么第一次执行`Thread.interrupted()`方法时候返回true，随后无论执行多少次`Thread.interrupted()`方法都返回false。

​		由此可知方法`parkAndCheckInterrupt()`可使在用户只调用一次`thread.interrupt()`方法的情况下，`if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())`中它自身只返回一次true。这样就不会重复执行满足if条件后的语句了。

```java 
private final boolean parkAndCheckInterrupt() {
    	// 阻塞线程，执行此方法后线程会阻塞在这里
        LockSupport.park(this);
    	// 当调用 LockSupport.unpark(Thread)或者thread.interrupt()方法时候
    	// 被阻塞的线程会从上面方法返回，因此可以继续执行下面程序
        return Thread.interrupted();
}
```

###### 4.2 解锁过程

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ReetrantLock/images/unLock.png)

​		① 调用`ReetrantLock.unlock()`方法，这个方法内部调用了`sync.release()`

```java
public void unlock() {
        sync.release(1);
}
```

​		② 本例中`sync.release()`方法实际是调用了AQS类中的`release()`方法

```java
public final boolean release(int arg) {
    	// 尝试释放锁
        if (tryRelease(arg)) {
            Node h = head;
            // 如果头不为空，则说明有等待队列
            // 若在第一个条件为true的条件下，waitStatus不为0说明还有后继节点需要唤醒
            if (h != null && h.waitStatus != 0)
                // 唤醒头结点的后继节点
                unparkSuccessor(h);
            return true;
        }
        return false;
}
```

​		③ 本例中tryRelease()实际调用的Reentrant.Sync.tryRelease()方法，此方法主要将同步器的状态进行变更

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ReetrantLock/images/wakeup.png)

```java
protected final boolean tryRelease(int releases) {
    	    // 当前状态减去释放值的结果赋值给c
            int c = getState() - releases;
    	    // 如果当前线程不是获取锁的线程，则抛出非法监视器状态异常 
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
    	   // 标记是否是无锁状态
            boolean free = false;
    		// 若c==0 则表示当前处于无锁状态
            if (c == 0) {
                free = true;
                // 设置获得锁的线程为Null
                setExclusiveOwnerThread(null);
            }
    	    // 设置锁状态值为c
            setState(c);
    	    // 返回free，若为true则证明完全释放锁，若有后继节点，可以唤醒后继节点了
    		// 若为false，则可能是重入锁的案例，锁还不能被其后继节点获取
            return free;
}
```

​		④ 调用AQS类下的`unparkSuccessor()`方法，此方法主要是处理好传入节点本身的状态，然后唤醒其后继节点。

```java
private void unparkSuccessor(Node node) {
        // 获取传入节点的waitStatus
        int ws = node.waitStatus;
    	// 如果waitStatus<0 那么使用CAS将node的状态设置为0
    	// 使用CAS是担心此时有其他节点修改传入节点的waitStatus
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        // 获取node节点的下一个节点
        Node s = node.next;
    	// 若下一个节点为空或者下一个节点waitStatus>0 (Cancelled状态)
        if (s == null || s.waitStatus > 0) {
            // 先将s置空（s!=null && waitStatus>0）
            s = null;
            // 从等待队列尾部开始遍历，找到一个可用的（waitStatus<=0）节点赋值给s
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
    	// 如果后继节点不为空，唤醒后继节点的线程
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

**总结** 

​		以上就是我个人理解的ReentrantLock的原理和核心方法，其他方法在以后的学习中会继续记录。以后每次学习后通过写成文字的方式再梳理一遍，希望能有所收获。如果我有新的理解也会不断迭代此文章。
