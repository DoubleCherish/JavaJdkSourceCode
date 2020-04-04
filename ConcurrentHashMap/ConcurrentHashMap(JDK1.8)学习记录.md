### ConcurrentHashMap(JDK1.8)学习记录

​		看了忘忘了看系列之ConcurrentHashMap，本文主要记录下通过看ConcurrentHashMap源码学习到的知识点。主要有以下几个点。文章稍长，需要耐心阅读。

​		1、ConcurrentHashMap构造函数和相关属性

​		2、ConcurrentHashMap使用示例

​		3、ConcurrentHashMap跟随示例学原理

​		ConcurrentHashMap的出现主要是因为HashMap在多线程情况下表现不好。那么下面文章就跟着源码学习下ConcurrentHashMap是如何在多线程下表现良好的。

##### 1、ConcurrentHashMap构造函数和相关属性

**构造函数**

​		ConcurrentHashMap的构造函数和HashMap的构造函数形式类似，但是在带容量参数的构造函数中调用tableSizeFor函数时候稍有不同

```java
// 无参构造函数，也是大家经常使用的
public ConcurrentHashMap() {
}

// 带容量参数的构造函数
public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   // 这里为什么要将传入的容量变为大于1.5倍容量+1的最小的2的整数次方幂
                   // 还有待理解
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
}


// 使用其他Map作为参数构造函数 
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this.sizeCtl = DEFAULT_CAPACITY;
        putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
}
```

**属性变量**

```java
// 下面列举一些重要的属性变量，许多变量和HashMap的还是挺相似的

// 最大table容量
private static final int MAXIMUM_CAPACITY = 1 << 30;

// 不带参数构造时候，当添加第一个元素时候默认扩容的table大小
private static final int DEFAULT_CAPACITY = 16;

// table单个槽位树化的阈值，槽位中链表节点树大于这个值才尝试树化
static final int TREEIFY_THRESHOLD = 8;

// 一般当扩容之后单个槽位的节点数量会变少，当节点数量少于这个值时候将树转为链表
static final int UNTREEIFY_THRESHOLD = 6;

// 树化节点时候需要table容量大于下面这个值
static final int MIN_TREEIFY_CAPACITY = 64;

// 单个线程最小处理步长
private static final int MIN_TRANSFER_STRIDE = 16;


private static int RESIZE_STAMP_BITS = 16;

private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
// 实际存储键值对的table
transient volatile Node<K,V>[] table;
// 扩容时候用到的引用
private transient volatile Node<K,V>[] nextTable;
// 暂时是table的键值对数量
private transient volatile long baseCount;
// sizeCtl：
// 大于0代表table.length的0.75倍
// -1代表正在初始化
// -N 代表有 -（N-1)个线程正在扩容
private transient volatile int sizeCtl;
// 扩容时候用到的记录转移索引
private transient volatile int transferIndex;
```

**静态方法**

```java
// 下面三个方法调用了Unsafe的一些操作内存的方法，用反射获取Unsafe的成员变量theUnsafe可以做实验

// 原子的获取table的i位置的元素
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
// CAS机制替换i位置的元素
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,Node<K,V> c, Node<K,V> v) {
     return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
CAS机制i位置元素赋值
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

##### 2、ConcurrentHashMap使用示例

​		下面我演示一个普通的用法，然后第三部分借这个用法示例分析ConcurrentHashMap核心方法

```java
public class MainTest {
    public static void main(String[] args) {
        ConcurrentHashMap<String,String> concurrentHashMap = new ConcurrentHashMap<>();
        // 这里是方便使得ConcurrentHashMap扩容触发
        for(int i=0;i<13;i++){
            concurrentHashMap.put("name"+i,"fuhang");
        }
        
        concurrentHashMap.get("name1");
        
        concurrentHashMap.remove("name1");
    }
```

##### 3、ConcurrentHashMap跟随示例学原理

​		这里借用我在记录HashMap时候画的table的简易结构图来说下ConcurrentHashMap的结构，两者基本结构类似。

![]()

​		构造函数已经在上面贴出，这里不再赘述。我直接从`put()`方法开始切入。

**put相关方法**

​		① put(K,V)

```java
// 对外暴露的put方法，ConcurrentHashMap内部调用putVal()方法进行实际处理
public V put(K key, V value) {
        return putVal(key, value, false);
}

// 实际的put方法，最后一个参数代表当旧值存在时候是否替换旧值
final V putVal(K key, V value, boolean onlyIfAbsent) {
    	// 在这里可以看出ConcurrentHashMap中的key和value都不能为空
    	// HashMap中的key可以为空，为空时候hash值为0
        if (key == null || value == null) throw new NullPointerException();
    	// 调用spread方法计算hash，后面分析
        int hash = spread(key.hashCode());
    	// binCount记录一个槽位中存在多少个node
        int binCount = 0;
    	// 下面开始for循环将值放入table
        for (Node<K,V>[] tab = table;;) {
            // f ： 定位到的节点
            // n ： table的长度
            // i :  hash%n算出的节点索引
            // fh： 代表定位到的节点的hash
            Node<K,V> f; int n, i, fh;
            // 如果tab为空或者长度为0，那就初始化table，大多是初次调用put方法时候执行的
            if (tab == null || (n = tab.length) == 0)
                // initTable()初始化方法在后面分析
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果定位到的槽位的node为null，那么直接CAS机制将key，value放入i位置
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                // 如果定位到的节点的hash值为MOVED（-1），则说明table正在扩容
                // 调用helpTransfer方法协助扩容
                tab = helpTransfer(tab, f);
            else {
                // 到此就说明没扩容，定位到的f节点不为空，说明有碰撞了
                // 通过拉链法将当前节点放入链尾部
                V oldVal = null;
                // 首先对链的头节点加锁
                synchronized (f) {
                    // 加锁之后需要再次判断i索引位置的节点是否为此次循环中的节点
                    // 因为可能在上一步判断完成之后，另个线程执行扩容恰好移动到了i位置
                    // 那么i位置的节点就由f变为fwd了，所以加锁之后需要再次判断定位的节点
                    // 是否和之前一样
                    if (tabAt(tab, i) == f) {
                        // 判断fh的hash大于0
                        if (fh >= 0) {
                            // 设置节点计数器为1
                            binCount = 1;
                            // 下面从头结点向后遍历
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果i位置的链表中存在一个节点的key和
                                // 当前给的key值相等，!onlyIfAbsent为真的情况下覆盖旧值
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    // 跳出循环
                                    break;
                                }
                                // 要不然往后面遍历
                                Node<K,V> pred = e;
                                // 若遍历到尾结点了，那就将当前新值追加到尾结点之后
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            // 如果 f 节点是一个树节点，那就执行红黑树插入
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // 如果i位置的节点数量不为0
                if (binCount != 0) {
                    // 如果 i 位置的节点数量大于等于8，就尝试将i位置的所有节点树化
                    // 在treeifyBin内部还需要满足table的长度>64才树化
                    // 要不然就尝试扩容
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    // 如果旧值存在，说明没有新的节点创建
                    // 那么久直接返回旧值，不需要程序再执行扩容检测
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 这里就是类似于HashMap中的resize()方法的作用
        addCount(1L, binCount);
        return null;
}
```

​		② spread(int h)

```java
// ConcurrentHashMap中算hash的方法
// HASH_BIT : 0111 1111 1111 1111 1111 1111 1111 1111
static final int spread(int h) {
    	// 这里和HashMap的比较多了一步
    	// 要将(h ^ (h >>> 16))的结果和HASH_BIT做与运算
    	// 这一步主要是为了将(h ^ (h >>> 16))的最高位置0，也就是保持hash为正数
    	// 因为hash的正负值也代表不同的逻辑意义
        return (h ^ (h >>> 16)) & HASH_BITS;
}

// 回顾下HashMap中的算Hash的方法
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

​		③ initTable()

​		这个方法一般是在第一次putVal时候调用，用来初始化table，使用sizeCtl来标志初始化状态，sizeCtl = -1表示正在初始化，初始化完成以后将sizeCtl设置成table容量的0.75倍，类似HashMap中的阈值作用。

```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
    	// while循环进行判断table是否初始化完成
        while ((tab = table) == null || tab.length == 0) {
            // 如果sizeCtl小于0，则让出cup占用权
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                // 要不然将将sizeCtl设置为-1，标志此时正在初始化
                try {
                    // 这里为什么要在做判断容我思考下
                    if ((tab = table) == null || tab.length == 0) {
                        // 若sizeCtl>0(初始化ConcurrentHashMap传入容量的情况) n=sc
                        // 要不然n=DEFAULT_CAPACITY (无参构造的情况)
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        // 创建Node数组，长度为n
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // sc设置为0.75 * n，类似HashMap中的loadFactor*Cap
                        sc = n - (n >>> 2);
                    }
                } finally {
                    // 将sizeCtl赋值为sc
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
}
```

​		④addCount(long x, int check)

```java
// 这个方法类似于HashMap中的resize()
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            // 这里先暂时不做分析，先记住if判断中将s赋值为baseCount+x
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
    	// 如果check>=0
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            // while循环判断扩容是否完成
            // 当s（可以理解为table中的元素数量）>sizeCtl（阈值）时候
            // 且table！=null 且 table.length<MAXIMUM_CAPACITY时尝试扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                // 根据n也就是table.length算一个rs，这个操作相当于一个扩容前的标记
                int rs = resizeStamp(n);
                // 如果sc小于0，说明正在扩容
                if (sc < 0) {
                    // 1、如果sc 绝对右移16位（相当于丢弃记录扩容线程的后16位）不等于rs
                    // 说明扩容标记有变化，本次扩容和之前的不是同一个扩容
                    // 2、如果sc==rs+1 说明扩容结束，因为sc+2是初始值，sc-1是扩容完成
                    // 3、如果sc == rs +MAX_RESIZER（1<<16）-1，说明扩容线程已到达最大
                    // 4、nt=nextTable==null 说明扩容结束
                    // 5、如果transferIndex<=0 也说明扩容结束
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    // 要不然将将sc+1(扩容线程数量+1)，开始扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    // sizeCtl不小于0，那就将sizeCtl设为(rs << RESIZE_STAMP_SHIFT) + 2)
                    // 在这个时候sizeCtl值的前16位表示扩容开始前将容量大小用二进制表示时候
                    // 从左往右数第一个1之前0的个数，相当于封存下扩容前的场景。后16位表示
                    // 有多少线程正在扩容，设置成功后sizeCtl就变成负数了
                    // 这里+2的操作就是当第一个线程扩容完成后会执行 rs-1操作，方便上面判断
                    // 调用transfer方法开始扩容
                    transfer(tab, null);
                // 统计table中的节点数量
                s = sumCount();
            }
        }
}
```

​		⑤ transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)

​		我在代码分析里面先介绍下transfer的逻辑，随后会用一个容量为16，sizeCtl为12的map来添加12个元素，实际分析一次扩容做了什么。

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    	// n 为原table的长度，stride记录处理步长
        int n = tab.length, stride;
    	// 计算步长
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 如果nexttab为空则说明是第一次调用扩容
    	if (nextTab == null) {            // initiating
            try {
                // 创建一个原表长度两倍大的新表nt
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                // 将nt赋值给nextTabl
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            // 将nextTab赋值给ConcurrentHashMap的成员变量nextTable
            // 将n赋值给ConcurrentHashMap的成员变量transferIndex
            // 回想一下addCount()方法中当sc小于0时候的第一个if判断
            nextTable = nextTab;
            transferIndex = n;
        }
    	// 用nextn记录新表的长度
        int nextn = nextTab.length;
    	// 创建一个ForwardingNode ，这是Node的子类，内部有一个成员变量为Node[] nextTable
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    	// 这个标志控制下面while循环
        boolean advance = true;
    	// 扩容结束标志
        boolean finishing = false; // to ensure sweep before committing nextTab
    	// for循环进入
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            // 若advance为真进入while循环做处理
            while (advance) {
                int nextIndex, nextBound;
                // 若 --i>=bound 说明扩容执行完成
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    // 如果nextIndex<=0则可以进行结束判断了
                    // 退出while循环进行结束检测
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    // 要不然在这里将transferIndex更新下
                    
                    // 记录边界值
                    // 记录开始转移的索引 i
                    // 将advance设为false以便退出while循环进行下面操作
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            // 结束判断
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 如果结束标记为真
                if (finishing) {
                    // 将成员变量nextTable设为null
                    // 回想下addCount()中当sc小于0的if判断
                    nextTable = null;
                    // 将成员变量table设为nextTab
                    table = nextTab;
                    // 记录sizeCtl为0.75 * 2n
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                
                // 若没结束，先用sc记录sizeCtl的值，再将sizeCtl设为sizeCtl-1
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 如果此时恢复到转换前的值不相等，则直接返回，说明还有扩容线程在运行
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 要不然将finish和advance标志设置为true
                    finishing = advance = true;
                    // 将i赋值为n是为了能重新再进入此代码块
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
               	// 如果扩容时候i位置的元素为null 那就将其CAS设置为fwd节点
                // 若设置成功则advance为true，方便进入上面while循环将i进行递减
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                // 如果i位置节点的hash值为MOVED，则说明是fwd节点
                // 说明已经执行过扩容处理。将advance设为true，进入while循环将i递减
                advance = true; // already processed
            else {
                // 要不然将f加锁进行节点转移
                synchronized (f) {
                    // 这里再次判断i位置的节点还是不是f
                    // 因为到这里才加锁进行操作，故多个线程可能同时执行到此
                    // 若其中一个线程瞬间加锁完成f节点的转移后释放锁
                    // 那么其他线程进行下面判断时候就失败了
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            // 这里我会画图做个解释，类似HashMap的操作
                            // ln代表在原位置不变的头结点
                            // hn代表扩容后在i+n位置的头结点
                            // 下面会有图示举例
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            // 标志为for循环1，方便图示中解释
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            // 标志为for循环2
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // 将nextTable i位置节点设置为ln
                            // 将nextTabl  i+n位置节点设为hn
                            // 将原tab的   i位置设为fwd节点表示已经转移
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            // 如果fh<0 则先判断下是不是红黑树树根节点
                            // 若是则进行树节点的转移
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            // 下面代码类似HashMap
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            // 如果高低位置有节点数量小于6的，就尝试取消树化
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
 }
```

​		以上就是transfer函数的代码分析，口述下过程就是：

​				1、先获取当前table的长度记为n，再根据CPU个数给线程分配步长，最小处理步长为16

​				2、如果传递过来的nextTab为null，则说明是第一个扩容线程，创建一个2n大小的nextTab，赋值给全局变量nextTable，并将n值赋值给全局变量transferIndex

​				3、使用nextn记录新表长度(2n)，创建一个ForwardingNode节点，原table每一个槽位转移完成都会暂时将槽位节点设置成ForwardingNode节点。

​				4、定义两个boolean变量advance和finishing，一个控制while循环调整索引，一个标志扩容是否结束

​				5、进入for循环开始扩容，首先进入advance控制的while循环对索引边界进行设置

​				6、紧接着if判断索引i是否满足一些条件，若满足进行结束判断

​				7、若(6)不满足的情况下，将原tab的 i 位置的节点赋值给f，判断f是否为空，若为空的话，使用CAS机制将i位置的节点设为ForwardingNode，并置advance为CAS设置节点的结果。

​				8、若(7)不满足的情况下，将f节点的hash值赋值给fh变量，若fh==MOVED(-1)，说明i位置节点已经在扩容过程中转移完成，将advance设为true，方便下一次循环执行

​				9、若(8)不满足的情况下，说明f节点为转移，则使用synchronized先对f节点加锁，然后判断i位置的节点是否还是f节点，若是则进入 f 节点扩容

**默认容量16，sizeCtl(阈值)为12，添加第12个元素时候的流程图**

![]()



**普通链表节点转移示意图**



​		⑥ helpTransfer(Node<K,V>[] tab, Node<K,V> f)

​		这个方法主要是感知到其他线程正在扩容，然后协助扩容的逻辑。

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
    	// 如果tab不为空，且f是ForwardingNode节点，且fwd节点的nextTab不为空
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            // 计算扩容标志
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                // 这个判断前面已经讲解，这里不再赘述
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                // 将扩容线程+1，调用transfer(tab,nextTab)协助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
}
```

​		⑦ treeifyBin(Node<K,V>[] tab, int index)

​		当某次put键值后的binCount>=8时候调用此方法，此方法中继续判断table.length若小于64则继续调用tryPresize()方法将table进行扩容。若table.length不小于64，则尝试将 index索引位置的节点进行树化

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
}
```

​		⑧ tryPresize(int size)

```java
// 此处是当table.length<64时候会调用到的一个方法
private final void tryPresize(int size) {
    	// 如果size(n<<1) 大于最大容量的一半，则将c设为最大容量
    	// 要不然将c设为大于（size*1.5+1）的最小的2的整数次幂值
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            // 下面这段代码有待思考，以后再来补充注释
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                // 下面这段就是一个正常扩容逻辑，在这里不再赘述
                int rs = resizeStamp(n);
                if (sc < 0) {
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
}
```

**get相关方法**

​		ConcurrentHashMap的get方法基本和HashMap的get方法类似

```java
public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

**总结**

​		以上就是暂时看完源码后对ConcurrentHashMap的理解，将理解用文字和图的形式记录下来，其中也有暂时没想通的点，随后想通后会回来继续补充完善文章。