### LinkedList源码学习记录

##### 0、简介

​		最近处于老项目上线，新项目设计初期，稍有空闲，于是开始记录下学了忘忘了学系列之LinkedList源码。本篇主要记录以下几个点。

​		1、LinkedList使用示例及优势

​		2、LinkedList构造函数及属性

​		3、LinkedList核心方法分析

##### 1、LinkedList使用示例

​		LinkedList底层基于双向链表的数据结构组织数据，理论上节点个数无上限（堆中放的下的情况下）。在插入删除较多的场景下，读相对较少的情况下效率高于ArrayList。

**示例**

```java
import java.util.LinkedList;
public class Test {
    public static void main(String[] args) {
        LinkedList<String> linkedList = new LinkedList<>();

        linkedList.add("fuhang");
        linkedList.add(0,"zhangsan");
        linkedList.addLast("lisi");

        linkedList.remove("lisi");

    }
}
```

##### 2、LinkedList构造函数及属性

**继承结构**

![](https://github.com/DoubleCherish/JavaJdkSourceCode/blob/master/LinkedList/image/LinkedList.png)

**构造函数**

```java
// LinkedList只有两个构造函数，带一个集合参数的和不带参数的
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
}
```

**属性**

```java
// 元素个数
transient int size = 0;
// 第一个元素的引用
transient Node<E> first;
// 最后一个元素的引用
transient Node<E> last;
// 节点类型
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
}
```

##### 3、LinkedList核心方法分析

​		下面根据示例作为小小的桃花源入口，然后看看桃花源内部有啥。

① add(E e)

​		此方法是大家最常用的方法，用于向LinkedList添加元素。具体代码如下

```java
public boolean add(E e) {
        //可以看出是尾插法
        linkLast(e);
        return true;
}
// 方法很简单，所以简单注释下
void linkLast(E e) {
    	// 获取尾结点
        final Node<E> l = last;
    	// 创建新节点，前一个节点设为尾结点，后一个为null
        final Node<E> newNode = new Node<>(l, e, null);
    	// 更新尾结点指针
        last = newNode;
    	// 如果之前尾结点为空，则将头结点设为新的节点，要不然将将之前尾结点的下一个节点
    	// 设为新创建节点
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
    	// 容量计数器和修改计数器都自增
        size++;
        modCount++;
 }
```

② remove(Object o)

​		此方法主要用来删除指定对象元素

```java
public boolean remove(Object o) {
    	// 如果给定待删元素为null
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            // 如果给定数据不为null
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
}

E unlink(Node<E> x) {
    	// 这里内部其他函数保证了x!=null
        // assert x != null;
    	// 拿到给定元素值、前后节点引用
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;
		// 如果prev == null，则说明删除头结点，那么更新头结点引用
    	// 要不然将prev.next赋值为x的下一个节点
        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }
		// 如果next为空，则说明删除的是尾结点，那么更新尾结点引用
    	// 要不然将下一个节点的前向引用更新为待删节点的前一个节点
        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }
		// 将x.item=null,方便GC
        x.item = null;
    	// 元素计数器自减1，修改次数自增1
        size--;
        modCount++;
        return element;
}
```

③ get(int index)

​		此方法主要用来获取给定位置的元素，在方法内部有头尾指针和元素计数器，使得程序可以判断给定索引处于链表前半段还是后半段，使得遍历次数最多为N/2

```java
public E get(int index) {
    	// 范围检查
        checkElementIndex(index);
    	// 获取给定索引位置元素值
        return node(index).item;
}

private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

private boolean isElementIndex(int index) {
        return index >= 0 && index < size;
}

Node<E> node(int index) {
         // assert isElementIndex(index);
	    // 判断index是在前半部分还是后半部分
    	// 根据index所处位置选择从头遍历还是从尾部遍历
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
}
```

**剩下方法一起分析**

**头部添加元素**

```java
public void addFirst(E e) {
    linkFirst(e);
}
// 方法如下
private void linkFirst(E e) {
		// 获取头结点
        final Node<E> f = first;
    	// 创建新节点
        final Node<E> newNode = new Node<>(null, e, f);
    	// 更新头结点引用
        first = newNode;
    	// 如果之前头结点为空，那么顺带将尾结点引用更新。
    	// 要不然将原先头结点的pre引用指向新创建节点
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
}
```

**指定位置添加元素**

```java
// 在指定位置添加元素
public void add(int index, E element) {
        checkPositionIndex(index);
		// 如果index大于size直接添加到尾部，在这里有个疑问，为什么判断了尾插不判断首插
        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
}

void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
    	// 获取即将成为后继节点的节点的前一个节点
        final Node<E> pred = succ.prev;
    	// 创建新节点并设定前后节点
        final Node<E> newNode = new Node<>(pred, e, succ);
    	// 让原后继节点的前一个节点指向新节点
        succ.prev = newNode;
    	// 如果原先后继节点的前一个为空，则更新头结点引用
    	// 要不然将前一个节点的next指向当前创建的新节点
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
   }
```

**删除指定位置节点**

```java
public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
}
```

**获取指定元素在链表中的位置O(N)**

```java
public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
}
```

**队列操作--获取队首元素**

​		这里只是抛砖引玉说下和队列数据结构相关的方法，更多方法和队列操作类似

```java
public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
}
```

**栈操作--弹出栈顶元素**

​		这里只是抛砖引玉说下和栈数据结构相关的方法，更多方法和栈操作类似

```java
public E poll() {
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
}
```

**总结**

​		由以上的分析可以知道LinkedList底层采用一个双向链表作为底层数据结构，从源码实现可以得出结论，我们可以将LinkedList当做链表、栈、队列、双端队列等多种数据结构来用。以上就是本次简单的学习记录。
