### HashMap（JDK1.8）源码剖析

​		这又是看了忘忘了看系列之一，今天有空写个文档记录下，希望能从JDK源码中慢慢悟出他们优秀的思想。本文主要记录以下几个方面。

​				1、HashMap的继承、实现结构

​				2、HashMap的构造函数们及属性们

​				3、HashMap的核心方法们

##### 1、HashMap的继承、实现结构

![](https://github.com/DoubleCherish/JavaJdkSourceCode/tree/master/HashMap(JDK1.8)/image/HashMap.png)

​		以上就是HashMap的继承结构图，相对来说是比较简单的结构。下面是HashMap的概念图。

![](https://github.com/DoubleCherish/JavaJdkSourceCode/tree/master/HashMap(JDK1.8)/image/table.png)

##### 2、HashMap的构造函数们及属性们

###### 2.1 属性

​		我们先看下HashMap的属性变量

```java

// 如果创建HashMap时候不指定容量，那么默认容量就是16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 最大容量为2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认容量因子，符合泊松分布
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 树化一个槽位的阈值，一个槽位节点个数大于这个值且满足容量大于最小容量就树化
static final int TREEIFY_THRESHOLD = 8;

// 树化还原阈值
static final int UNTREEIFY_THRESHOLD = 6;

// 最小树化容量值
static final int MIN_TREEIFY_CAPACITY = 64;

// 实际存放节点的table
transient Node<K,V>[] table;

// 方便遍历所有节点的entrySet
transient Set<Map.Entry<K,V>> entrySet;

// 节点实际个数
transient int size;

// 结构变化次数，删除新增节点都会导致结构修改
transient int modCount;

// 扩容阈值，size大于这个值就扩容，已使HashMap的效率得到保障
int threshold;

// 容量因子，可手动指定，默认为0.75f
final float loadFactor;
```

###### 2.2 构造函数

​		看下HashMap的构造函数

```java
// 我们最常使用到的构造函数
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

// 带容量参数的构造函数
public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

// 带容量参数和容量因子
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
    	// 根据容量算个阈值
        this.threshold = tableSizeFor(initialCapacity);
}

// 带其他Map集合参数的构造方法
public HashMap(Map<? extends K, ? extends V> m) {
    	// 容量因子设为默认值0.75f
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        // 将其他Map的所有节点放入table
        putMapEntries(m, false);
}
```

##### 3、HashMap的核心方法们

​		我先举个HashMap常用的例子，跟着例子一步一步深入到HashMap所有方法。

```java
public class MainTest {
    public static void main(String[] args){
        HashMap<String,String> keyValues = new HashMap<>();
        final String name = "name";
        keyValues.put(name,"fuhang");
        keyValues.get(name);
        keyValues.remove(name);
    }
}
```

​		以上就是一个简单的使用示例，因为构造函数在（2.2）节分析过，由此可知上述使用构造时候只是初始化了下容量因子。下面我们逐个方法深入分析。**注** : 红黑树操作暂不分析，有空专门记录红黑树数据结构。

###### 3.1 put(K key,V value)

```java
// 这是我们最常调用的put方法
public V put(K key, V value) {
    	// 内部调用putVal方法
        return putVal(hash(key), key, value, false, true);
}
```

###### 3.2 hash(Object key)

```java
// 这个方法是HashMap内部的静态工具方法，将key的hashCode和其绝对右移16位后的值做异或运算
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

![](https://github.com/DoubleCherish/JavaJdkSourceCode/tree/master/HashMap(JDK1.8)/image/hashcode_or.png)

​		HashMap使用key的hashcode和key的hashcode绝对右移16位做异或运算后作为key新的hash值是为了减少key的碰撞做的优化，将高位数据移动到地位参与运算。

###### 3.3 tableSizeFor()

```java
// 此方法是将给定的容量扩大到恰好刚刚大于给定容量的2的整数次幂倍，方便以后定位元素位置
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

图示如下：

![](https://github.com/DoubleCherish/JavaJdkSourceCode/tree/master/HashMap(JDK1.8)/image/tableSize2.png)

###### 3.4 putVal(...args)

```java
putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict){
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    	// 如果table为空或者table里面没元素，table结构会在后面有图形示例
        if ((tab = table) == null || (n = tab.length) == 0)
            // 调用resize()方法给table扩容，将table.length赋值给n
            n = (tab = resize()).length;
    	// 如果根据key的hashcode和table的长度计算出来的位置上元素为空，
    	// 那么直接把元素放到此位置
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            // 要不然就是计算出来的位置上已经有元素出现碰撞了，采用拉链法添加元素
            Node<K,V> e; K k;
            // 判断第一个元素是否和key的hash或者值相等
            // 若相等，将p赋值给e
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 第一个元素不等后，判断第一个元素是不是树节点
            // 若是的话进行节点插入树结构
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 第一个元素不是树节点，且第一个元素key和我们要添加的key比较不等于
                // 则执行以下逻辑
                // 循环i位置的链节点，类似于遍历链表
                for (int binCount = 0; ; ++binCount) {
                    // 如果遍历到最后p.next为空了，就直接在链的尾部p.next添加节点
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 如果在i位置遍历次数大于8，那么尝试树化i位置的所有节点
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 如果p.next不为空，那么判断此节点的key是否和我们put的相等
                    // 若相等就跳出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // 上面判断都不成立就继续向后面遍历
                    p = e;
                }
            }
            // 如果e不为空，说明table已经存现相同key值，若onlyIfAbsent为false
            // 那么就使用新值覆盖旧值，返回旧值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
    	// 执行到此还没有返回的话，就确实是一个新节点被添加到table中了
    	// 修改记录自增，size自增并判断是否大于阈值，大于则扩容
        ++modCount;
        if (++size > threshold)
            resize();
    	// 这个方法在HashMap中为空，是在节点插入后可以做一些响应
        afterNodeInsertion(evict);
        return null;
}
// Callbacks to allow LinkedHashMap post-actions
 void afterNodeAccess(Node<K,V> p) { }
 void afterNodeInsertion(boolean evict) { }
 void afterNodeRemoval(Node<K,V> p) { }
```

以上就是put方法的分析，其中有几个小点下面解释下：

​		① `i = (n - 1) & hash` 这个等式相当于` i = hash % n`，但是前者效率更高，HashMap就是为了高效所以做到细节优化，**注** ：上面等式成立的前提是 n 必须是 2 的整数次幂。

![](https://github.com/DoubleCherish/JavaJdkSourceCode/tree/master/HashMap(JDK1.8)/image/&&.png)

​		② onlyIfAbsent这个标志为true时候，若key所对应的值已经存在，那就不会使用新value覆盖旧value。若为false时候，会使用新value覆盖旧的value

​		③ ` binCount >= TREEIFY_THRESHOLD - 1` 时候只是可能会将table[i]位置对应的链进行树化，其方法内部还要满足` tab.length >= MIN_TREEIFY_CAPACITY` 才进行树化

###### 3.5 get(Object key)

```java
// HashMap对外暴露的get方法
public V get(Object key) {
        Node<K,V> e;
    	// 内部调用getNode进行获取
        return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    	// 首先得判断 table 不为空 且table长度大于 0 且key所对应位置的第一个元素不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 判断first是不是就是我们要找的元素，若是直接返回first
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            // 如果first不是，就准备遍历对应位置的链表
            if ((e = first.next) != null) {
                // 如果第一个位置的元素是树节点，说明此位置已经树化
                // 那么调用红黑树的getTreeNode进行查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 不是树化，且first不是要找的节点，就遍历对应位置的链结构数据逐个查找
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
}
```

###### 3.6 remove(Object key)

```java
// 从HashMap里面移除一个key对应的node
public V remove(Object key) {
        Node<K,V> e;
    	// 内部调用removeNode进行移除，成功返回移除key对应的值，失败返回null
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}

/**
     * Implements Map.remove and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to match if matchValue, else ignored
     * @param matchValue if true only remove if value is equal
     * @param movable if false do not move other nodes while removing
     * @return the node, or null if none
 */
// 参数意义如上注释
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
    	// 这个判断个getNode中的一样，先要找到key对应的node，这里不再啰嗦
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            // 若找到key对应的node了，移动其next指针就行，类似链表操作
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    // 红黑树移除，移除满足一定条件就将树退化为链表
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

###### 3.7 resize()

```java
 final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
     	// 获取旧table容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
     	// 旧的阈值
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 如果旧table容量大于等于最大容量限制
            // 那么将阈值更新为Integer的最大值2147483647并返回老表
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 要不然将新容量扩大为老table的 2 倍
            // 若同时满足老table容量大于初始容量的条件，就将阈值也扩大到原来的 2 倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }else if (oldThr > 0) // initial capacity was placed in threshold
            // 这个if条件是在调用带容量参数或者容量参数和容量因子的构造函数构造的HashMap
            // 在随后第一次调用resize()方法执行的代码
            // 将新容量赋值为老阈值的值
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 这个是无参或者参数为Map的构造函数第一次进入resize方法时候执行的代码
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
     	// 如果newThr值为0
     	// 1、可能进入 if (oldCap > 0) 但是 oldCap >= DEFAULT_INITIAL_CAPACITY条件不满足
     	// 2、可能进入 if (oldThr > 0)
        if (newThr == 0) {
            // 根据新容量*容量因子计算ft
            float ft = (float)newCap * loadFactor;
            // 如果新容量小于最大容量，且ft小于最大容量，那么就将ft赋值给新阈值
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
     	// 将新阈值赋值给成员变量threshold
        threshold = newThr;
     	// 创建新的table,并将其赋值给成员变量table
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
     	// 如果老的table不为空，说明里面有元素，那么就要将元素移入新table
        if (oldTab != null) {
            // 从头遍历老table
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                // 如果老table的j位置不为空，那么就要将元素迁移到新table
                if ((e = oldTab[j]) != null) {
                    // 由于上面将j位置的node赋值给e，下面可以置空利于垃圾收集器回收
                    oldTab[j] = null;
                    // 如果e的下一个节点为空，那么直接将e迁移到新表对应位置就行
                    // 位置只能是原位置或者原位置+oldCapacity位置
                    // eg. 6%4=2  扩容2倍后   6%8=6
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 如果e是树节点，那么将调用树的split方法将其重新迁移到新table中
                    // 这个方法也可能将树退化为链表
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 到这里说明节点是正常的拉链节点了
                        
                        // loHead和loTail用于迁移在原位置不动的节点们
                        Node<K,V> loHead = null, loTail = null;
                        
                        // hiHead和hiTail用于迁移在原位置+oldCap的节点们
                        Node<K,V> hiHead = null, hiTail = null;
                        
                        // 遍历链表使用的临时变量next
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 原节点的hash & oldCap为0的节点保持原位置
                            // 这个计算可以理解为原hash % newCap == hash % oldCap
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                // 要不然就在原位置+oldCap位置
                                // 这里就是原hash % newCap != hash % oldCap的
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 最后判断高位地位的尾部指针是否为空
                        // 不为空说明有节点迁移，那么将相应的头节点赋值给相应位置
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
 }
```

以上就是resize()方法的所有逻辑，一般来说都是将容量和其阈值一起扩大为原先的2倍，在做数据迁移就完成了。其中有几个点在下面解释下。

​			① 我起初第一次看代码时候不理解` if ((e.hash & oldCap) == 0) `这个判断的意思，经过计算后就显而易见，举例如下图。

![](https://github.com/DoubleCherish/JavaJdkSourceCode/tree/master/HashMap(JDK1.8)/image/e_hash&oldCap.png)

**总结**:

​		以上即是HashMap常用方法的分析，其中关于红黑树部分的代码暂时没分析，等到下次记录TreeMap时候将红黑树结构一起做分析。
