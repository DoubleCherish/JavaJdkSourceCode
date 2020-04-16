### Java线程池学习记录（一）

##### 1、引言

​        今天做线程池学习记录是在写web3j调用以太坊进行压测代码的时候想起来的，从毕业到现在工作已经1年多了，这期间来来回回看过线程池代码好几遍，结果都是看了后忘记一部分，再看再忘记。以前看别人看了东西都会记录下来，自己没这个习惯，借此机会希望自己以后养成记录的习惯。

##### 2、概要目录

* 示例代码
* Executor
* ExecutorService
* Executors
* ThreadPoolExecutor
* AbstractExecutorService
* Worker
* ThreadPoolExecutor下execute()、addWorker()、runWorker()、getTask()、processWorkerExit()

##### 3、源码分析

​        在分析源码之前先看张类之间的关系图，再看一个简单使用案例作为预备知识。
![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ThreadPool/image/one.jpg)

```java
public class Main {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                System.out.println("");
            }
        });
    }
}
```

​        一个简单的使用案例如上代码所示，创建一个线程池然后向线程池提交任务，下面我们就逐步分析其中的代码

##### Executor

```java
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

​        Executor接口就定义了一个方法`execute(Runnable command)`， 这个方法在未来某个时间执行给定的任务，这个任务也许在一个新的线程中被执行，也许在缓存的线程中执行，也许会在其灵活的实现类中被调用执行。

##### ExecutorService

```java
public interface ExecutorService extends Executor {

    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();

    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
    
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

​        由上面内容可以看到ExecutorService继承自Executor，在ExecutorService中有定义了许多方法，主要分为提交线程相关方法、停掉线程池相关方法、几个用于执行给定任务的方法。

##### Executors

​	    这个类好似一个工厂，里面存在各种静态方法，主要用来创建各种类型的线程池，下面我们就其中一个newFixedThreadPool()来做分析

```java
public class Executors {
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
}
```

​		 这个方法就是创建一个线程数量可控的线程池，现在公司里面使用线程池大多都自己定义，自己定义可以控制很多参数。很少使用Executors的方法直接创建

##### AbstractExecutorService

​       `AbstractExecutorService`是一个抽象类，它实现了`ExecutorService` 接口中的部分方法，下面只介绍其中一部分示例代码中使用的方法

```java
public abstract class AbstractExecutorService implements ExecutorService {
    // 由protected修饰可以看出这个方法可以由子类实现
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        // FutureTask实现了RunnableFuture
        return new FutureTask<T>(runnable, value);
    }
    // 这个方法就是我们示例代码中调用的方法
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        // 使用newTashFor方法将Runnable包装成一个RunnableFuture
        // RunnableFuture是一个接口继承了Runnable和Future
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        // 紧接着调用实现类的execute方法
        execute(ftask);
        return ftask;
    }
}
```

##### ThreadPoolExecutor

​        这个类是平常使用到的重点类，核心的方法都在这个类里面定义，接下来我们分析其核心要点

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ThreadPool/image/state_threadpool.png)

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    // 线程状态标志器
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 线程最大容量 1^29 个
    // CAPACITY = 0001 1111 1111 1111 1111 1111 1111 1111
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
	
    // 用一个Integer变量，4字节的前3位来控制线程状态，后面29位控制线程数量
    // runState is stored in the high-order bits
    // (二进制表示) running = 1010 0000 0000 0000 0000 0000 0000 0000
    private static final int RUNNING    = -1 << COUNT_BITS;
    // SHUTDOWN = 0000 0000 0000 0000 0000 0000 0000 0000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // STOP = 0010 0000 0000 0000 0000 0000 0000 0000
    private static final int STOP       =  1 << COUNT_BITS;
    // TIDYING = 0100 0000 0000 0000 0000 0000 0000 0000
    private static final int TIDYING    =  2 << COUNT_BITS;
    // TERMINATED = 0110 0000 0000 0000 0000 0000 0000 0000
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    // 算出当前线程池状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 计算当前worker个数
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    // 算状态
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    
    private final BlockingQueue<Runnable> workQueue;
	
    private final HashSet<Worker> workers = new HashSet<Worker>();
    private int largestPoolSize;
    private volatile long keepAliveTime;
    private volatile boolean allowCoreThreadTimeOut;
    private volatile int corePoolSize;
    private volatile int maximumPoolSize;
    
    
    // 选最终的构造函数看一看
    // 参数从前到后分别是：
    //					核心池大小
    //					最大池大小
    //					线程存活时间(这个在特定情况下才生效)
    // 					时间单位
    //  				阻塞队列（有几种不同的队列，后面介绍）
    //  				线程工厂（默认的给线程添加一些标记，也可以自己实现）
    //  				拒绝策略（在不能创建线程时候的处理方法）
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

​        以上就是ThreadPoolExecutor的主要成员变量及构造方法，接下来我们回顾我们的示例代码

```java
public class Main {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newFixedThreadPool(10);
	    // 接下来分析submit方法，这个方法是在AbstractExecutorService中被实现的
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                System.out.println("");
            }
        });
    }
}
```

​	    由示例代码可以看出我们调用了`submit()` 方法，此方法做的事情如上面分析所示，下面分析一下核心方法`execute()`

![2](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/ThreadPool/image/two.png)

```java
 // 下面开始看对Executor中execute方法在ThreadPoolExecutor中的实现
     public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *     如果有小于corePoolSize的worker在运行，那么试图创建一个新的worker执行任务
      	 * 调用addWorker方法时候会自动检查运行状态、线程数量，返回false警告时候
      	 * worker不应该被添加到线程池
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *    如果一个任务被成功添加到队列，我们仍然需要检查我们是否应该添加这个线程（因		    		
	 * 为从上次检查到现在线程池状态可能已经改变）因此我们重新检查状态，必要时进行回		
         *  滚。或者在入队成功并且worker为0时开启一个新线程
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         *    如果我们不能让任务进入队列，那么我们尝试开启添加一个新的线程去执行任务，
         * 如果失败，那么我们知道线程池是关闭了或者饱和了，因此我们拒绝这个任务
         */
	// 控制字段获取
        int c = ctl.get();
        // 策略1 
        if (workerCountOf(c) < corePoolSize) {
            // 这里在addWorker()方法里面检查了状态
            // 这里addWorker()失败的原因：
            // 1、进入方法后线程池关闭了，状态转为>=SHUTDOWN
            // 2、进入方法后别人修改状态，使得workerCountOf(c)>corePoolSize了
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 此处是策略2，线程运行情况下，将任务加入队列
	// 这个队列有好几种选择，有容量为Integer.MAX_VALUE的也有容量为0的队列。
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 这里线程池是运行状态检查为非运行以后，remove()失败了，那感觉还是会执行			 				    // 任务，随后再理解下
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }else if (!addWorker(command, false)) // 如果加入队列失败，则再调用一次尝试将woker数量扩大到maximumPoolSize
            	reject(command);	      // addWorker()，策略3	
    }
    
    // 下面我们分析一下addWorker方法
    // 参数 firstTask：要执行的任务 core:是否是核心线程下添加
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            // 获取线程池状态
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 如果线程池的状态大于等于SHUTDOWN，意思是要关闭的情况
            // 且以下几种情况需要有一个为假的情况下就返回
            // 1、rs>=SHUTDOWN这个条件满足，且rs != SHUTDOWN的情况下就返回，因为线程			   				    // 池此时可能即将要关闭了，不能添加新的任务了
            // 2、rs==SHUTDOWN这个条件满足，并且要是firstTask != null那就返回，因为
            // 此时SHUTDOWN后会拒绝新任务提交。要是firstTask==null的情况下还可以继		   			   
	    // 续，这是线程池在SHUTDOWN情况下创建线程执行完剩余的任务，
            // 3、rs==SHUTDOWN && firstTask == null的条件下 workQueue.isEmpty()为			             
	    // 真的条件下返回，因为queue为空的情况下就不需要再创建新线程去执行任务了
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
		   // 如下for循环主要将线程池线程数量计数器加1
            for (;;) {
                // 计算线程池线程个数
                int wc = workerCountOf(c);
                // 如果线程个数大于最大容量1^29限制就返回，或者线程在core标记为真时，			            
		// 大于corePoolSize就返回，core为假时大于maximumPoolSize返回
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 调用compareAndIncrementWorkerCount将ctl数值+1，若成功就跳出retry				            
		// 循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 上面增加线程计数器失败，重新获取ctl，判断当前线程池状态是否和进入方                           
		// 法时一样(可能在进入方法后另个线程池改变了状态)，若不一样跳到retry处				       
		// 重新执行
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
	// 通过上面状态参数被正确修改以后，下面线程即将被真正创建并执行
        // 下面两个标记位标志线程是否开启、worker是否被添加
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 创建一个Worker，参数传入我们需要执行的任务，后面分析Worker
            w = new Worker(firstTask);
            // 拿到Worker的成员变量thread
            final Thread t = w.thread;
            if (t != null) {
                // 拿到当前线程池的锁，添加worker时候需要加锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    // 获取当前线程状态
                    int rs = runStateOf(ctl.get());
		    // 如果线程状态是Runing，或者SHUTDOWN且firstTask==null为真时候
                    // （SHUTDOWN && firstTask==null代表线程池已经关闭，开启线程执行  				                                 // 完线程池在关闭之前添加的任务）
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // workers是一个HashSet<Worker>
                        workers.add(w);
                        int s = workers.size();
                        // largestPoolSize标记线程池大小的Peek值
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 如果线程被正确添加，那么开始线程，下面我们分析Worker的执行过程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
}
```

execute()总结一下就是：

​        1、当worker数小于corePoolSize时候将任务附着在一个worker上，创建worker同时执行任务。

​		2、当worker数量大于corePoolSize的时候，将不再创建新的worker，尝试将任务添加到工作队列workQueue当中，交由已有的worker去执行。

​		3、当workQueue工作队列里面放满task时候，那么尝试去扩展worker数量到maximumPoolSize个，这个要是失败就拒绝新的任务

​		当线程池被shutdown()后，线程池会确保有一个worker或者去执行任务队列里剩下的任务。`execute`和`addworker` 方法分析完成以后，下面我们紧接着分析worker是如何执行任务的

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
    {
        private static final long serialVersionUID = 6138294804551838833L;
	// 可以看出worker里面组合了一个Thread和一个Runnable成员变量
        final Thread thread;
        Runnable firstTask;
        volatile long completedTasks;

        Worker(Runnable firstTask) {
            // 构造时候设置状态为-1，禁止中断直到运行runWorker
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask
            // 使用worker自身作为Thread的参数创建线程，为代理调用做准备
            this.thread = getThreadFactory().newThread(this);
        }

        // 这里相当于一个代理调用，学到这里又会从jdk源码里面学到一招
        // 上面addWorker方法里面调用了worker.thread.start()后会调用这里这个run方法
        // 这个run方法会调用runWorker将自身当参数传入，随后分析runWorker()
        public void run() {
            // runWorker是ThreadPoolExecutor的方法
            runWorker(this);
        }

        // 以下是状态锁相关的代码，用来控制线程中断等逻辑。
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
        	// 每次tryAcquire都是状态从0-1，故是一个不可重入锁
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }
	// 这个方法中，只有当线程state>=0才可中断
        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```

​        以上就是Worker类，是ThreadPoolExecutor类的一个内部类，实现了Runnable，继承自AQS，用到了AQS本身的一些不可重入锁机制，自身run代理调用了runWorker方法以自身为参数做一些事情，下面我们分析runWorker方法

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
    	// 拿到worker自身携带的任务，可能为null
        Runnable task = w.firstTask;
        w.firstTask = null;
    	// 解锁worker自身锁，接受中断请求
        w.unlock(); // allow interrupts
    	// 是否突然完成标记
        boolean completedAbruptly = true;
        try {
            // 下面这个while循环就是所谓的主循环，getTask方法随后会分析一下，主要是从
            // 队列里面获取任务
            while (task != null || (task = getTask()) != null) {
                // 加锁是为了在线程池调用shutdown()方法时候不能中断运行任务的线程
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                // 1、如果线程状态>=STOP且当前线程中断标志为false，那么中断当前线程
                // 2、如果线程状态<STOP，调用Thread.interrupted()方法获取当前线程
                //  中断标志并清除标志，若Thread.interrupted()返回true说明当前线程已			   		                 //  经中断，且再次获取状态>=STOP且当前线程中断状态标志为false，那么中				       
		// 断当前线程
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                // 上面判断通过后开始执行下面流程
                try {
                    // 这个方法没有实现，可以在任务开始前执行一些操作
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 实际调用task.run()方法运行用户自定义任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 这个方法也没实现，用户自己实现，可在任务执行完以后做操作
                        afterExecute(task, thrown);
                    }
                } finally {
		   // 在上面流程执行完毕以后，将task置空方便垃圾回器回收、将当前					 
		   // worker执行完任务数+1、对当前worker进行解锁
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            // 将是否非正常完成标记设为false
            completedAbruptly = false;
        } finally {
            // 最后处理Worker退出工作
            processWorkerExit(w, completedAbruptly);
        }
    }
```

​		以上就是runWorker的执行流程，其中的while循环是其主循环，一个worker在这个while循环中会执行自身任务，若自身任务执行完毕就调用getTask从队列里面获取任务执行，当没有任务需要执行时候执行处理worker退出流程。下面我们分析下getTask()和processWorkerExit()方法。

```java
private Runnable getTask() {
    	// 超时标志
        boolean timedOut = false; 

        for (;;) {
            // 获取当前线程池状态
            int c = ctl.get();
            int rs = runStateOf(c);

            // 如果 线程池状态>=SHURDOWN && 状态>=STOP
            // 或者 线程池状态>=SHUTDOWN && workerQueue为空
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                // 减少worker数量并返回null，这个方法不成功减少worker计数不返回
                // 会一直循环cas减少worker数量直到成功
                decrementWorkerCount();
                return null;
            }
		   // 如果上面if不满足则说明线程还是running状态，下面先获取worker数量
            int wc = workerCountOf(c);

            // Are workers subject to culling?
            // 是否设置了允许核心线程超时或者worker数量已经大于corePoolSize
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		   // 1、如果worker数量大于maximumPoolSize且worker数量大于1，将worker计数器
            // 减少1返回，这种情况就是在addWorker时候成功，随后用户调用了					   
	    // setMaximunPoolSize()
            // 2、如果worker数量大于maximumPoolSize且workerQueue为空，将worker计数器
            // 减少1返回，若减少worker计数器失败，则表明状态已经变化，跳过此次循环
            // 3、若timed为true且timeOut为true且worker数量大于1，将worker计数器
            // 减少1返回，若减少worker计数器失败，则表明状态已经变化，跳过此次循环
            // 4、若timed为true且timeOut为true且workerQueue为空，将worker计数器
            // 减少1返回，若减少worker计数器失败，则表明状态已经变化，跳过此次循环
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    // worker数量大于corePoolSize时调用下面方法
                    // 这个方法指定时间获取不到结果就会返回null
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                	// worker数量小于corePoolSize时候直接调用这个方法获取任务，
                	// 这个方法会一直卡在这里等待任务，有任务才返回，底层调用了Condition.await()阻塞线程，等待Condition.signal()来唤醒			        // 由此可推断在这里阻塞线程，当workQueue里有元素时候会调用condition.signal()来唤醒阻塞的线程
                    workQueue.take();
                // 如果任务不为空，则返回任务
                if (r != null)
                    return r;
                // 获取任务超时标记为true
                timedOut = true;
            } catch (InterruptedException retry) {
                // 如果线程被中断了，超时标志设置为false
                timedOut = false;
            }
        }
    }
```

上面getTask里面有几个点值得记录一下：

​        1、getTask方法刚刚进入的时候回判断线程池状态，如果线程池状态>=SHUTDOWN且状态>=STOP，那么说明可能是shutdownNow()被调用了，线程池需要停止，所以不处理任务。

​        2、如果线程池状态>=SHUTDOWN，但是还不到STOP，且workerQueue为空，则说明到达SHUTDOWN状态且任务队列为空，那么没必要处理任务了，直接返回。

​        3、timed一般常用于标志worker总数是否大于corePoolSize，若大于corePoolSize的情况下，timed就为true，那么在下面就会Runnable r 获取的时候会调用workQueue,poll(time,unit)，这个方法在指定时间获取不到任务就返回空(当线程数量大于corePoolSize的情况下，timed为true，且workQueue中没有任务的情况下workQueue.poll()会返回空，后面timeOut=true成立)导致一系列条件成立，timed&&timeOut则为true，workQueue.isEmpty()为true，因此会调用减少worker的方法并返回。

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 如果runWorker时候，快速完成标记为true，说明运行时出现异常，那么worker
        // 计数器-1
    	if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
    	// 获取主锁，将worker完成的任务数量加到线程池完成任务数量计数器中
    	// 将worker从worker set中移除
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
	    // 检查是不是可以让线程终止，后面分析
        tryTerminate();
	    // 获取线程状态
        int c = ctl.get();
    	// 如果状态<STOP,可能是SHUTDOWN或者RUNNING，判断要不要添加worker，要不然
    	// 什么都不做
        if (runStateLessThan(c, STOP)) {
            // 如果非快速完成，就是worker正常执行完工作，没有事情可做时候
            if (!completedAbruptly) {
                // 取最小worker数量值
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                // 如果min==0且workQueue工作队列不为空，那么最少需要存在一个worker
                // 去完成剩下的任务，设置min=1
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // 如果worker数量已经大于min了就直接返回，要不然调用addWorker添加				   
		// worker
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
	
	// 这个方法是用来判断线程池是不是可以终止
	final void tryTerminate() {
        for (;;) {
            // 获取线程池状态
            int c = ctl.get();
            // 1、如果线程池处于运行状态，不需要终止，那么返回。
            // 2、如果线程池状态>=TIDYING，那么返回，说明线程池已经关闭或者正在关闭
            // 3、如果线程池状态==SHUTDOWN且workQueue不为空，那么返回，得等到			   			   	     //    workQueue里面的任务执行完毕才能终止
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 以上条件都不满足，那么判断worker不为0个的情况下尝试中断一个空闲worker
            // 并返回
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
		   // 如果worker数量为0的情况下
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 将线程池状态设置为TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        // 调用terminated()方法终止线程池，这个方法在
                        // ThreadPoolExecutor中是个空实现，可以看出这个方法是
                        // 交给各个子类自己实现的
                        terminated();
                    } finally {
                        // 将线程池状态设置为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        // termination是mainLock的Condition
                        // 用于唤醒等待此条件的任务
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // 设置不成功则重新执行for循环，说明在方法运行期间状态有变化
            // else retry on failed CAS
        }
    }
```

​	以上就是线程池的大部分方法的实现逻辑，随后会再分析下线程池调用shutdown等方法的逻辑。

