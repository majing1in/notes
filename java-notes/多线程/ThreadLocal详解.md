[TOC]

## 一、ThreadLocal是什么

Thread Local为线程提供自己的变量副本，当多个线程访问一个公共变量时为了保证了安全性，一般分为两种方式，第一种方式就采用以时间换空间的方法来加锁实现，第二种就是采用Thread Local以空间换时间的方式，为每个线程创建一个属于自己的本地变量。

## 二、ThreadLocal、Thread、ThreadLocalMap之间的关系

ThreadLocalMap是ThreadLocal得内部类（key ->ThreadLocal，value -> Object）；

在Thread中拥有ThreadLocalMap变量，并且使用缺省修饰，没有提供直接访问方式，只能通过ThreadLocal进行操作，使线程与变量操作进行分离；

ThreadLocal操作ThreadLocalMap的对象，并且是ThreadLocalMap的key；

综上：一个Thread程对应一个ThreadLocalMap，ThreadLocalMap保存着当前Thread的各种ThreadLocal。

## 三、ThreadLocal源码分析

#### 1. set方法

```java
public void set(T value) {
    // 获取当前的Thread线程
    Thread t = Thread.currentThread();
    // 获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 将当前ThreadLocal作为key，保存到ThreadLocalMap中
        map.set(this, value);
    else
        // 初始化ThreadLocalMap
        createMap(t, value);
}
```

#### 2. get方法

```java
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 利用当前ThreadLocal作为key获取value
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 初始化ThreadLocalMap或者直接通过ThreadLocalMap设置value
    return setInitialValue();
}
```

#### 3. remove方法

```java
public void remove() {
    // 获取当前线程的ThreadLocalMap
     ThreadLocalMap m = getMap(Thread.currentThread());
     if (m != null)
         // 移除当前ThreadLocalMap中的ThreadLocal对象
         m.remove(this);
 }
```

## 四、ThreadLocalMap源码分析

#### 1. 初始化ThreadLocalMap

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 默认数组大小16
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```



