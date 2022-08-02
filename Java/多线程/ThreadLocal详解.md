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

获取在数组中的位置，判断当前位置是否为空，如果不为空则发生了hash冲突

进入方法中判断当前key与当前位置上的key是否是同一个，如果是则更新value，如果当前位置的key为null，可以判断已经被会后或者已确定不再使用，会对它进行回收；

如果当前key并不是同一个，则会向后探索空位，然后进行插入；

最后会清理一些未使用的槽位，判断是否需要扩容。

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

#### 3. replaceStaleEntry替换陈旧的key

在set方法中插入时，当前数组上的key存在但是已经被清理了，会进行替换；

首先从当前位置staleSlot向前进行环形搜索，直到遇到entry中的空位置停下，记录下标slotToExpunge；

从staleSlot向后环形搜索，对每一个数组上的key进行判断，如果之前有相同key放入了数组，但是因为hash冲突而导致了后移，需要将它换回到原位，防止同一个数组中存在两个相同的key；

如果确实已存在key替换完成后，对数组进行清理；

不存在则清空当前位置的value，并将新的key放入到当前位置。

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
    int slotToExpunge = staleSlot;
    // 向前环形搜索，寻找是否存在被回收的key
    for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;
    // 向后环形搜索，寻找因为hash冲突而后移并且是相同的key，替换回原有位置
    // 保证一个数组中不存在相同的两个key
    for (int i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 找到被hash冲突后移的key，交换回原位
        if (k == key) {
            e.value = value;
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            // 删除被清理的key，调整后续key的位置
            // 重被清理放回的位置开始局部扫描清理
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

#### 4. expungeStaleEntry删除陈旧的key

根据传入的下标位置进行向后环形搜索；

当探索过程中entry不为空时，会判断当前entry的key是否为空，如果为空则代表不在使用，进行删除；不为空则会获取它原本在数组中的位置，与当前位置进行判断，如果与原位置不相同，则置空当前位置，最好情况还原到该出现的位置，否则向后探索空位进行插入，优化查询效率；

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // 当前的key已经被清理了，直接置空
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    Entry e;
    int i;
    // 向后探索，寻找是否存在已经被清理的key，当遇到第一个为null的entry则返回下标
    for (i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 删除已经被回收的key
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // 原本在数组中的位置
            int h = k.threadLocalHashCode & (len - 1);、
            // 因为hash冲突导致位置发生变化，将当前所在位置清理，重新寻找空位放入
            if (h != i) {
                tab[i] = null;
                // 最好情况回归原位，否则向后探索空位并放入
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

#### 5. cleanSomeSlots清理为空的key

从当前位置向后环形扫描，在(n >>>= 1) != 0的范围次数内找到key为null的位置，进行删除并重置扫描次数与扫描位置。

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        // 向后探索清理被回收的key
        i = nextIndex(i, len);
        Entry e = tab[i];
        // 找出被清理的key，找到了则重新从i位置继续探索，并重置扫描次数
        if (e != null && e.get() == null) {
            // 重置扫描次数
            n = len;
            removed = true;
            // 删除陈旧的key，返回扫描位置
            i = expungeStaleEntry(i);
        }
        // 限制扫描次数(范围)
    } while ((n >>>= 1) != 0);
    return removed;
}
```

#### 6. getEntry方法

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 获取应在所在数组的下标
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 找到直接放回entry
    if (e != null && e.get() == key)
        return e;
    else
        // 向后探索寻找目标并返回
        return getEntryAfterMiss(key, i, e);
}
```

#### 7. getEntryAfterMiss方法

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    // 向后探索，因为是遇到hash冲入会向后寻找空位并放入
    // 如果后一位被清理，会删除当前位置，重写计算后面不为null的key的位置
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            // 有被清理的key进行删除，并会调整后续key的位置
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

#### 8. rehash方法

```java
private void rehash() {
    // 删除数组中已经被回收的key
    expungeStaleEntries();
    // 实际保存容量达到数组一半时进行扩容 1/2
    if (size >= threshold - threshold / 4)
        resize();
}
```

#### 9. expungeStaleEntries删除过时的条目

```java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    // 全局扫描删除被回收的key
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

#### 10. resize扩容方法

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    // 扩容原有的2倍
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
    // 对原有数组进行全局扫描
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; 
            } else {
                // 计算在新数组中的位置
                int h = k.threadLocalHashCode & (newLen - 1);
                // 发生hash冲入向后探索放入
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
    // 设置阈值
    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

## 五、ThreadLocal弱引用设计

https://zhuanlan.zhihu.com/p/139214244

![ThreadLocal_1](D:\notes\java\资源\ThreadLocal_1.png)

总的来说Threadlocal的设计是对线程本地变量的操作，将线程本地变量与维护的关系进行了分离，直接使用定义的ThreadLocal即可，所以Thread没有直接对ThreadlocalMap操作的方法；

ThreadLocalMap是线程局部变量，生命周期与线程一致，当线程销毁ThreadLocalMap也会一并销毁；

ThreadLocalMap本身并没有为外界提供取出和存放数据的API，我们所能获得数据的方式只有通过ThreadLocal类提供的API来间接的从ThreadLocalMap取出数据，当我们用不了ThreadLocal对象的API也就无法从ThreadLocalMap里取出指定的数据；

如果ThreadLocal在entry中是强引用，即使ThreadLocal Ref被置为空，根据GC Root进行扫描规则，依旧存在一条从entry.key指向ThreadLocal的强引用，但是Thread无法直接操作ThreadLocalMap，只有线程结束才能正确的释放；

而如果使用弱引用当ThreadLocal Ref被置为空，entry.key就没有了直接关联的强引用对象，进行垃圾回收的时候会被回收掉，当key被回收了，就可以利用key是否为null进行对entry的删除。

## 六、ThreadLocal内存泄漏原因

对于ThreadLocalMap而言时Thread的成员变量，如果线程结束则ThreadLocalMap也会被销毁，这种情况不存在内存泄漏；

但是存在线程复用(线程池)的情况下，ThreadLocalMap会永远存在Thread中，如果ThreadLocal被回收了，此时ThreadLocalMap中的key就为null，而value就无法被访问，造成了内存泄漏；

在ThreadLocal中所有的set、get、remove方法都会检测数组中为null的key进行删除，但是还是推荐在使用了set方法后手动remove，防止内存泄漏。
