### HashSet学习记录

##### 0、简介

​		本篇主要记录下HashSet的源码学习记录，是随机看到的，随手记录下。大家都知道HashMap是一种存储键值对的数据结构，HashSet从名字可以看出来其主要是一个集合，通过数学上集合的概念可以知道集合存放非是没有重复数据的集。HashSet底层实现是通过组合了一个HashMap实现的，通过借助HashMap中key不能重复的特性实现HashSet想要的功能。

​		本篇学习记录主要记录以下几个点:

​						1、HashSet的使用示例

​						2、HashSet的构造方法及属性

​						3、根据示例看其方法源码

##### 1、HashSet的使用示例

​		HashSet是一个没有重复元素的集合，演示代码如下：

```java
public class HashSetTest {

    public static void main(String[] args) {
        HashSet<String> set = new HashSet<>();
        set.add("fuhang");
        set.add("zhangsan");
        set.add("lisi");
        set.add("fuhang");
        System.out.println(set.size());
        set.contains("fuhang");
        set.remove("fuhang");

    }
}
```

##### 2、HashSet的构造函数及属性

**继承关系图**

![HashSet](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/HashSet/HashSet.png)

**属性**

```java
// 内部组合一个HashMap
private transient HashMap<E,Object> map;

// 这个为了填充HashMap的value创建的统一对象
private static final Object PRESENT = new Object();
```

**构造函数**

```java
 // 无参构造函数，内部创建了一个HashMap()，调用了HashMap的无参构造函数
 public HashSet() {
        map = new HashMap<>();
 }

// 参数为另一个集合的构造函数
public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
 }

 // 参数为初始容量及负载因子的构造函数
 public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
 }

 // 参数为初始容量的构造函数
 public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
 }

 // 同包下其他类可调用的构造函数，底层创建了一个LinkedHashMap
 HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
}

```

##### 3、根据示例看其方法源码

​		下面逐个方法开始分析。[HashMap](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/HashMap(JDK1.8)/HashMap%EF%BC%88JDK1.8%EF%BC%89%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90.md)源码可以看我另一篇详细的分析文章。

​		① add(E e)

​		add方法内部调用了HashMap.put()(一般是HashMap)方法，使用e作为key，固定对象PRESENT作为value

```java
// 内部调用HashMap的put方法
public boolean add(E e) {
        return map.put(e, PRESENT)==null;
}
```

​		② remove(Object o)

​			底层调用HashMap的remove方法

```java
public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
}
```

​		③ contains(Object o)

​			底层还是调用HashMap的方法

```java
public boolean contains(Object o) {
        return map.containsKey(o);
}
```

**总结**

​		以上就是很简单的HashSet"内部"，希望以后用到时候无疑惑。
