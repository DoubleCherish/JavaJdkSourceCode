### 根据一致性Hash算法学习TreeMap



##### 0、简介

​		本次根据使用TreeMap实现简单一致性Hash算法的例子来学习下这个让人忘了学学了忘的TreeMap源码，希望从中学到它的设计思想。本次主要记录以下几个点。

​		1、TreeMap实现一致性Hash算法示例

​		2、TreeMap构造函数及属性

​		3、TreeMap核心方法分析

##### 1、TreeMap实现一致性Hash算法示例

​		简介下算法内容，先构造一个长度为Integer.MAX_VALUE长度的一个整数环(一致性Hash环)，根据节点的Hash值将服务器放在Hash环上。然后根据数据的Key值计算其Hash值，接着在Hash环上顺序查找距离这个Key值最近的服务器节点，完成Key到服务器的映射。

​		由于java本身的hashCode值分散不够均匀，所以使用FNV1_32_HASH算法替换java对象自身的hashCode方法来计算key的hash值。

​	下面先看下一个使用TreeMap实现一致性Hash算法的示例。

```java
import java.util.SortedMap;
import java.util.TreeMap;

public class ConsistentHashing {

    private static String [] servers = {"127.0.0.1:9002","127.0.0.1:9003","127.0.0.1:9004"};

    private static TreeMap<Integer,String> serverList = new TreeMap<>();

    static {
        for(int i=0;i<servers.length;i++){
            int hash = getHash(servers[i]);
            serverList.put(hash,servers[i]);
        }
    }

    private static int getHash(String str) {
        final int p = 16777619;

        int hash = (int) 2166136261L;

        for (int i = 0; i < str.length(); i++)
            hash = (hash ^ str.charAt(i)) * p;

        hash += hash << 13;

        hash ^= hash >> 7;

        hash += hash << 3;

        hash ^= hash >> 17;

        hash += hash << 5;

        if (hash < 0)
            hash = Math.abs(hash);

        return hash;
    }

    private static String getServer(String node){
        int hash = getHash(node);
        SortedMap<Integer,String> subMap = serverList.tailMap(hash);

        if(subMap == null || subMap.size()<=0){
            subMap = serverList;
        }
        return subMap.get(subMap.firstKey());
    }

    public static void main(String[] args) {
        String [] nodes = { "127.0.0.1:1111", "221.226.0.1:2222"};

        for(String node : nodes){
            System.out.println("node hash:"+getHash(node)+"路由服务器:"+getServer(node));
        }
    }
}
```

​		以上是一个简易的一致性Hash算法，其中用到了TreeMap一些特性，下面我们开始分析TreeMap。

##### 2、TreeMap构造函数及属性

**构造函数**

```java
// 无参构造函数
public TreeMap() {
        comparator = null;
}

// 带Comparator的构造函数
public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
}

// 带其他Map的构造函数
public TreeMap(Map<? extends K, ? extends V> m) {
        comparator = null;
        putAll(m);
}

// 带其他排序Map的构造函数
public TreeMap(SortedMap<K, ? extends V> m) {
        comparator = m.comparator();
        try {
            buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
        } catch (java.io.IOException cannotHappen) {
        } catch (ClassNotFoundException cannotHappen) {
        }
}
```

**属性**

```java
// 比较器，用于比较key键
private final Comparator<? super K> comparator;
// 红黑树的根节点
private transient Entry<K,V> root;
// 元素个数
private transient int size = 0;
// 修改次数
private transient int modCount = 0;
// TreeMap内部红黑树节点
static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;
}
```

**类继承结构**

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/TreeMap/images/TreeMap.png)

##### 3、TreeMap核心方法分析

​		按照惯例分析下TreeMap的常用核心方法。

**put(K key, V value)**

​		由下面分析可以得出TreeMap对元素数量没有限制，可以无限添加。

```java
public V put(K key, V value) {
    	// 获取红黑树根节点
        Entry<K,V> t = root;
    	// 如果根节点为空则创建根节点并返回null
        if (t == null) {
            compare(key, key); // type (and possibly null) check
            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // 获取当前的比较器
        Comparator<? super K> cpr = comparator;
    	// 如果比较器存在
        if (cpr != null) {
            do {
                // 获取红黑树根节点
                parent = t;
                // 然后根据比较结果定位元素位置
                // 若元素key已经存在，则更新其值，要不然结束循环
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {
            // 如果比较器不存在，且key为空则抛出空指针异常
            if (key == null)
                throw new NullPointerException();
            // 这块要求若没有传入比较器，那么key必须是实现Comparable接口，要不然出错
            @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                // 定位key所在内容
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
    
    	// 如果上面都没定位到，则创建一个新的节点放入key\value值，并设置父节点
        Entry<K,V> e = new Entry<>(key, value, parent);
    	// 按照查找树的性质，若给定key比父节点小，则是父节点的左子节点
    	// 若给定key比父节点key大，则放入父节点的右子节点
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
    	// 在插入节点后需要调用调整红黑树结构的方法
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }
```

**get(Object key)**

```java
public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
}

final Entry<K,V> getEntry(Object key) {
        // 判断是否有比较器，若有则调用另一个方法获取指定key的值
        if (comparator != null)
            return getEntryUsingComparator(key);
    	// 若没有比较器则key值不能为空，为空则抛出空指针异常
        if (key == null)
            throw new NullPointerException();
        // 下面key若未实现Comparable接口则会抛出异常
    	@SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    	// 下面就按照二插搜索树的方法来查找元素
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
        return null;
}

// 有比较器时候调用的查找元素方法
final Entry<K,V> getEntryUsingComparator(Object key) {
        @SuppressWarnings("unchecked")
        K k = (K) key;
    	// 获取比较器
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            // 按照二插搜索树的方式查找元素
            Entry<K,V> p = root;
            while (p != null) {
                int cmp = cpr.compare(k, p.key);
                if (cmp < 0)
                    p = p.left;
                else if (cmp > 0)
                    p = p.right;
                else
                    return p;
            }
        }
        return null;
}
```

**remove(Object key)**

```java
// 移除指定元素
public V remove(Object key) {
    	// 获取指定的Entry，方法如上面分析所示
        Entry<K,V> p = getEntry(key);
        if (p == null)
            return null;
	    // 如果指定元素存在，则在删除元素后返回其旧值
        V oldValue = p.value;
    	// 调用此方法函数指定元素
        deleteEntry(p);
        return oldValue;
}


private void deleteEntry(Entry<K,V> p) {
        modCount++;
        size--;

        // If strictly internal, copy successor's element to p and then make p
        // point to successor.
        if (p.left != null && p.right != null) {
            Entry<K,V> s = successor(p);
            p.key = s.key;
            p.value = s.value;
            p = s;
        } // p has 2 children

        // Start fixup at replacement node, if it exists.
        Entry<K,V> replacement = (p.left != null ? p.left : p.right);

        if (replacement != null) {
            // Link replacement to parent
            replacement.parent = p.parent;
            if (p.parent == null)
                root = replacement;
            else if (p == p.parent.left)
                p.parent.left  = replacement;
            else
                p.parent.right = replacement;

            // Null out links so they are OK to use by fixAfterDeletion.
            p.left = p.right = p.parent = null;

            // Fix replacement
            if (p.color == BLACK)
                fixAfterDeletion(replacement);
        } else if (p.parent == null) { // return if we are the only node.
            root = null;
        } else { //  No children. Use self as phantom replacement and unlink.
            if (p.color == BLACK)
                fixAfterDeletion(p);

            if (p.parent != null) {
                if (p == p.parent.left)
                    p.parent.left = null;
                else if (p == p.parent.right)
                    p.parent.right = null;
                p.parent = null;
            }
        }
}
```

**未完待续。。。**

