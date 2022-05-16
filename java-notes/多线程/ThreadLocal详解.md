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
    // threshold = len * 2 / 3;
    setThreshold(INITIAL_CAPACITY);
}
```

#### 2. set方法

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // 发生hash冲突会一直向后探索空位(环形) nextIndex => (return ((i + 1 < len) ? i + 1 : 0);)
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 已存在ThreadLocal直接赋值
        if (k == key) {
            e.value = value;
            return;
        }
        // 当前key已经被清理，替换陈旧的entry
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 将当前key插入到空位中
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

#### 3. replaceStaleEntry方法(替换陈旧的key)

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
    int slotToExpunge = staleSlot;
    // 向前环形搜索，找出遇到的第一个被回收的ThreadLocal对象
    for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;
    // 向后环形搜索
    for (int i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 线性探测解决hash冲突，向后探索找到如果被后移的key就将它替换到当前位置
        if (k == key) {
            e.value = value;
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }
    // 不存在被后移的key，
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

#### 4. expungeStaleEntry方法(删除陈旧的key)

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    Entry e;
    int i;
    // i = (e = tab[i]) != null
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 删除已经被回收的key
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // 重新计算ThreadLocalHashCode放入到新位置
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

#### 5. cleanSomeSlots方法(清理为空的key)

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        // 找出被清理的key
        if (e != null && e.get() == null) {
            // 重置扫描次数
            n = len;
            removed = true;
            // 删除陈旧的key，返回扫描位置
            i = expungeStaleEntry(i);
        }
        // 限制扫描次数
    } while ((n >>>= 1) != 0);
    return removed;
}
```



