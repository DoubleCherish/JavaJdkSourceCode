### Simplest Java  LruCache

##### 0、背景

​		风控规定不许在内存中缓存用户私钥等敏感数据，故原本产线的每次加密请求都需要去连接风控加密机获取私钥，现在为了提高性能和减少风控加密机压力，允许缓存少量热点商户账户私钥，脑海中第一想法就是做一个Lru缓存，于是做了下面最简单的一版，随后又新的需求再迭代修改。

##### 1、实现

​		先简单实现一个可以控制大小的lru缓存，底层基于LinkedHashMap，实现如下：

```java
package com.fu;

import java.io.IOException;
import java.util.LinkedHashMap;
import java.util.Map;

public class LruCache<K,V> extends LinkedHashMap<K,V> {

    int maxSize;

    public LruCache(int maxSize){
        super(8,0.75f,true);
        this.maxSize = maxSize;
    }

    public LruCache(int maxSize,int initialCapacity){
        super(initialCapacity,0.75f,true);
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        if(size() >= maxSize){
            return true;
        }
        return false;
    }

    public synchronized V put(K key, V value) {
        return super.put(key, value);
    }


    public static void main(String[] args) throws IOException {
        LruCache<String,String> lruCache = new LruCache<>(16);
        for(int i=0;i<10;i++){
            new Thread(new TestThread(lruCache)).start();
        }
        for(int i=0;i<30;i++){
            System.out.println(lruCache.size());
        }
        System.in.read();
    }
}

class TestThread implements Runnable{

    LruCache<String,String> lruCache;

    public TestThread(LruCache<String,String> cache){
        this.lruCache = cache;
    }

    @Override
    public void run() {
        for(int i=0;i<100;i++){
            lruCache.put(Thread.currentThread().getName()+i,"test");
        }
    }
}

```

![]()

#标签#随手记