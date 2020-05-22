### CyclicBarrier学习记录

##### 0、简介

​		此篇文章主要记录下忘了学学了忘系列之CyclicBarrier源码学习记录。夹缝里面抽时间记录，今天开发系统，联调被拖着，所以随便记录下。此篇文章主要记录以下几个点。

​		1、CyclicBarrier使用示例

​		2、CyclicBarrier构造方法及属性

​		3、CyclicBarrier核心方法分析

##### 1、CyclicBarrier使用示例

​		下面来看一个CyclicBarrier的使用示例。主要是模拟当所有人都到达时候开始上课的场景。

```java
import java.io.IOException;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CyclicBarrierTest {


    public static void main(String[] args) throws IOException {
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        CyclicBarrier cyclicBarrier = new CyclicBarrier(3,new Signal());
        Customer customerFuhang = new Customer("fuhang",cyclicBarrier);
        Customer customerZhangsan = new Customer("zhangsan",cyclicBarrier);
        Customer customerLisi = new Customer("zhangsan",cyclicBarrier);

        executorService.execute(customerFuhang);
        executorService.execute(customerZhangsan);
        executorService.execute(customerLisi);

        System.in.read();


        Customer customerWang = new Customer("wang",cyclicBarrier);
        Customer customerDong = new Customer("dong",cyclicBarrier);
        Customer customerFang = new Customer("fang",cyclicBarrier);

        executorService.execute(customerWang);
        executorService.execute(customerDong);
        executorService.execute(customerFang);

        System.in.read();

    }

    static class Signal implements Runnable{

        @Override
        public void run() {
            System.out.println("全体集合完毕，开始上课");
        }
    }


    static class Customer implements Runnable{
        private String name;
        private CyclicBarrier cyclicBarrier;
        public Customer(String name,CyclicBarrier cyclicBarrier){
            this.name = name;
            this.cyclicBarrier = cyclicBarrier;
        }
        @Override
        public void run() {
            System.out.println(name+"已到达");
            try {
                cyclicBarrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```

输出结果：

![]()

##### 2、CyclicBarrier构造方法及属性

**属性**

```java
    private static class Generation {
        boolean broken = false;
    }

    // 主锁
    private final ReentrantLock lock = new ReentrantLock();
    // 等待条件，等着被触发
    private final Condition trip = lock.newCondition();
    // 代表初始获取方个数
    private final int parties;
    // 到达等待条件后需要执行的命令
    private final Runnable barrierCommand;
    // 当前代
    private Generation generation = new Generation();
	// 这个是当前计数器，为0唤醒等待条件
	private int count;
```

**构造方法**

```java
 // 带一个初始数量的构造函数
 public CyclicBarrier(int parties) {
        this(parties, null);
 }
 // 带初始数量和满足条件后要执行的动作
 public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
 }
```

##### 3、CyclicBarrier核心方法分析

​		根据上面的示例，下面来分析其用到的方法。

**await()**

​		此方法可以理解为当线程执行到这里的时候若还不符合等待条件，那么久等待，要不然调用此方法唤醒所有等待线程。

```java
public int await() throws InterruptedException, BrokenBarrierException {
        try {
            // 使用调用dowait()方法，可以说是最核心的方法
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
}

private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        // 先获取主锁，然后加锁再操作
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 获取到锁以后，获取当前代
            final Generation g = generation;
		   // 如果当前代已经被打破，则抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
		   // 如果当前线程被中断，则设计打破屏障标志位，抛出中断异常
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
		    // 将count自减1，
            int index = --count;
            if (index == 0) {  // tripped
                // 如果index被递减到0了，那么准备执行给定的命令，并开启下一轮
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }
		    // 如果计数器不为0，则进入如下"死循环"
            for (;;) {
                try {
                    // 如果未设置过期，则调用condition.wait()等待
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }
			   // 当被别的线程唤醒后会执行下面逻辑
                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            // 锁释放
            lock.unlock();
        }
    }
```

**nextGeneration()**

```java
private void nextGeneration() {
        // signal completion of last generation
        trip.signalAll();
        // set up next generation
        count = parties;
        generation = new Generation();
}
```

**breakBarrier()**

```java
private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
}
```

**其他方法**

```java
public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
}

public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
}
```

