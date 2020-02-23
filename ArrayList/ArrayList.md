### ArrayList 源码手记

​		从大学到现在ArrayList源码看了忘忘了看，今天有空记录下自己看的东西以加深印象。下面开始按照自己的想法开始分析ArrayList源码。

​		1、先记录下ArrayList的类继承关系图



​		2、下面记录下ArrayList所实现接口中的三个标签接口，大部分人只熟悉第一个标签接口

```java
//先看下ArrayList的定义形式，实现了四个接口，其中后三个为标签接口
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, 	Cloneable, java.io.Serializable
{
}
// 标签接口一，大家最熟悉的序列化接口
public interface Serializable {
}
// 标签接口二，随机访问接口，暗示这个类支持快速（通常是常量时间）随机访问
public interface RandomAccess {
}
// 标签接口三，克隆接口
public interface Cloneable {
}
```

​		3、再来看下ArrayList的属性有哪些

```java
	// 默认初始化容量，若是无参构造的ArrayList，在添加第一个元素时候会将容量膨胀到下面大小
    private static final int DEFAULT_CAPACITY = 10;
	// 这个元素数组是静态的，那么全局唯一资源，在指定容量为0时或者其他几个场景会让elementData赋值为此数组
    private static final Object[] EMPTY_ELEMENTDATA = {};
	// 这个数组按照我个人理解，类似一个标记数组，当无参化构造了ArrayList后，会将elementData赋值为DEFAULTCAPACITY_EMPTY_ELEMENTDATA，在调用ArrayList的add方法添加第一个元素时候会判断element是不是和DEFAULTCAPACITY_EMPTY_ELEMENTDATA是同一引用，若是，就尝试将数组快速膨胀到DEFAULT_CAPACITY容量大小
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
	// 这个是实际存储元素的数组
    transient Object[] elementData; // non-private to simplify nested class access
	// 这个字段是实际元素的个数
    private int size;
	// 最大数组大小是2147483647 - 8 个 
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

​		4、接下来看看ArrayList的构造函数们

```java
// 构造方法一，无参构造方法，将elementData赋值为DEFAULTCAPACITY_EMPTY_ELEMENTDATA
public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    	//为什么只有在无参构造函数时才将elementData赋值为DEFAULTCAPACITY_EMPTY_ELEMENTDATA还暂未理解，猜想可能是带参构造函数里面的容量不能随意扩容，以防改变使用者原意。
}

// 构造方法二，带指定容量参数的构造方法
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            // 在这里若容量指定为0的时候，把elementData赋值为EMPTY_ELEMENTDATA
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
        }
 }

// 构造函数三，带集合参数的构造方法
public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 在这里若集合参数内容为空，则让elementData也指向EMPTY_ELEMENTDATA
            this.elementData = EMPTY_ELEMENTDATA;
        }
 }
```

​		5、ArrayList 函数分析，先从常用的函数分析

​				5.1 add(E e)

```java
// 先分析add方法，add方法将给定元素添加到elementData末尾，返回值为boolean类型
public boolean add(E e) {
    	// 第一步先验证容量和统计修改次数
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // 末尾添加给定值,size自增1
    	elementData[size++] = e;
        return true;
}

private void ensureCapacityInternal(int minCapacity) {
    	// 这里有两个函数调用，下面逐一分析
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

// 计算容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    	// 在这一步可以看出，若elementData和DEFAULTCAPACITY_EMPTY_ELEMENTDATA是同一引用，那么在添加第一个元素时候minCapacity就是1，肯定返回 DEFAULT_CAPACITY
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
}
private void ensureExplicitCapacity(int minCapacity) {
        // 修改次数加1
    	modCount++;
        // 当minCapacity大于实际数组长度时候扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
}

// 扩容代码
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
    	// 新容量为实际元素长度的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
    	// 如果新的容量还小于minCapacity，那就让新容量等于minCapacity，如果minCapacity值溢出上界，那么下面这个判断就不会执行。还有好几种情况如两个都溢出等等。
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 最后将调用Arrays.copy将生成一个新容量大小的数组，并将原来数组值放入，底层调用System.copy(src,srcPos,des,desPos,length)方法
        elementData = Arrays.copyOf(elementData, newCapacity);
}
```

​				5.2 add(int index, E element)

```java
// 在指定索引位置添加元素
public void add(int index, E element) {
        rangeCheckForAdd(index);
		// 容量检查，在上面已经分析过
        ensureCapacityInternal(size + 1);  // Increments modCount!!
    	// 这段就不多说了，将index位置及以后所有元素统一向后挪一个位置
        System.arraycopy(elementData, index, elementData, index + 1,size - index);
        elementData[index] = element;
        size++;
}

// add方法的范围检查
private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

​				5.3 addAll(Collection<? extends E> c) 、addAll(int index, Collection<? extends E> c)

```java
// 将一个集合所有元素添加到ArrayList中
public boolean addAll(Collection<? extends E> c) {
    	// 调用collection的通用方法toArray将集合转为数组
        Object[] a = c.toArray();
    	// 获取数组的长度，并进行扩容判断
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
}
```

​				5.4 get(int index)

```java
// 第二个比较常用的方法就是获取元素的方法get
public E get(int index) {
    	// 先做索引范围检查
        rangeCheck(index);
    	// 然后返回索引位置元素
        return elementData(index);
}
private void rangeCheck(int index) {
    	// 如果索引大于实际数组长度就抛异常
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
E elementData(int index) {
        return (E) elementData[index];
}
```

​				5.5 contains(Object o)

```java
// 个人感觉第三常用的方法就是用contains判断一个元素是否存在数组中
public boolean contains(Object o) {
        return indexOf(o) >= 0;
}

//这个方法是找元素在数组中首次出现的索引，被contains使用
public int indexOf(Object o) {
    	// 如果给定值为null，那么遍历数组元素看哪个元素的值为null
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            // 若给定值 != null 遍历数组逐个对比
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
    	// 没找到返回-1
        return -1;
}

// 这个是查找元素最后一次出现的索引值，这个就是从数组最后面的元素向前遍历
public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
 }
```

​				5.6 remove(int index)

```java
// 下来比较常用的方法就是remove方法，移除指定位置的元素
public E remove(int index) {
    	// index范围检查
        rangeCheck(index);
    	// 修改次数自增一
        modCount++;
    	// 获取要移除索引位置的元素
        E oldValue = elementData(index);
		// 总共要移动的元素的数量
        int numMoved = size - index - 1;
    	// 如果要移动的数量大于0，那么index之后的所有元素向前移一个位置
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,numMoved);
        elementData[--size] = null; // clear to let GC do its work
		// 返回要移除的元素
        return oldValue;
}

// 这个方法是移除指定元素，因为要判断数组里面是否存在指定元素，所以算法复杂度O(n)
public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    // 先找到指定元素的索引，如果找到索引位置了，那么移除时候就不需要索引范围检查，所以叫fastRemove
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
}

// 省略范围检查的步骤，直接移动数组，删除指定位置元素
private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        elementData[--size] = null; // clear to let GC do its work
}
```

​		以上就是ArrayList的主要方法，在ArrayList内部还有一些不常用的方法和内部类。现在可以看出ArrayList内部基于一个对象数组存放元素，因此拥有数组的特性: 定位快，但是查找需要O(n)复杂度，删除需要挪动的元素较多。