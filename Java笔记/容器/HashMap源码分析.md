### 一、位运算知识

1. &与运算两个位都是1时，结果才为1，否则为0

```tex
	1 0 0 1 1
&   1 1 0 0 1
————————————————
    1 0 0 0 1
```

2. |或运算两个位都是0时，结果才为0，否则为1

```tex
	1 0 0 1 1
|   1 1 0 0 1
————————————————
    1 1 0 1 1
```

3. ^异或运算，两个位上的值不同则为1，否则为0

```tex
	1 0 0 1 1
^   1 1 0 0 1
————————————————
    0 1 0 1 0
```

4. ~取反运算，0则变为1，1则变为0

```tex
~   1 1 0 0 1
————————————————
    0 0 1 1 0
```

5. <<左移运算，向左进行移位操作，高位丢弃，低位补0

```tex
9 << 3;
移位前：0000 0000 0000 0000 0000 0000 0000 1001
移位后：0000 0000 0000 0000 0000 0000 0100 1000
```

6. \>>右移运算，向右进行移位操作，对无符号数，高位补0，对于有符号数，高位补符号位

```tex
9 >> 3;
移位前：0000 0000 0000 0000 0000 0000 0000 1001
移位后：0000 0000 0000 0000 0000 0000 0000 0001
```

### 二、无符号右移16位后做异或运算

```java
// key如果是null 新hashcode是0否则计算新的hashcode
(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
```

1. ^按位异或运算，只要位不同结果为1，不然结果为0；>>>无符号右移：右边补0。key为Null的时候，hash算法最后的值以0来计算，也就是放在数组的第一个位置。
2. 将h无符号右移16为相当于将高区16位移动到了低区的16位，再与原HashCode做异或运算，可以将高低位二进制特征混合起来，如果直接用HashCode来进行索引算法的话，那进行运算的无论前面的怎么变，只有后四位参与运算，这样就会产生大量的Hash碰撞。
3. 采用^异或运算而不采用&与运算、|或运算的原因是^异或运算能更好的保留各部分的特征，如果采用&运算计算出来的值会向0靠拢，采用|运算计算出来的值会向1靠拢，采用&运算每一位的0或1的概率都是50%，使分布更加均匀。

### 三、为什么槽位数必须使用2^n

目的：为了让哈希后的结果更加均匀。

```java
// 计算当前key在数组中的位置
(n - 1) & hash
```

1. 如果length为2的幂次方，则length-1转化为二进制必定是11111……的形式，再同hashcode做&与操作，能充分保证了hashcode与length在二进制操作下，每个位置上最终结果在1与0之间的概率都是50%，从而使最终在table数组中分布更加均匀。
2. 进一步分析，如果不采用2的幂次方作为长度，则会浪费很大空间，例如以length=15为例，在1、3、5、7、9、11、13、15这八处不会存放数据，因为hashcode值在与14（即1110）进行&与运算时，得到的结果最后一位永远都是0，即0001、0011、0101、0111、1001、1011、1101、1111位置处是不可能存储数据的。
3. 在数组进行扩容的时候也充分使用了length为2次方幂的特性，在length等于16（即10000）的时候第1位是作为扩容的标准，它可以保证首位以后的数都为0，当第一位做&操作为1的时候就进行移动，否则不移动。

![HashMap-长度](D:\notes\Java笔记\资源\HashMap-长度.png)

HashMap构造函数允许用户传入的容量不是2的n次方，因为它可以自动地将传入的容量转换为2的n次方。会取大于或等于这个数的且最近的2次幂作为table数组的初始容量。

```java
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

### 四、HashMap的put方法流程

![HashMap-流程](D:\notes\Java笔记\资源\HashMap-流程.png)

1. 首先根据key的值计算hash值，并用当前的hash值进行高16位于低16位的异或运算得到最终的hash值，当key为null时则直接取值为0；
2. 当向map中存放元素的时候，如果使用默认的无参构造创建map，则会先对map进行初始化，则调用resize进行初始化；
3. 初始化完成后根据length-1与hash做与运算的得到元素在数组中的位置；
4. 如果位置上不存在元素则直接创建节点，如果存在节点则代表出现了hash冲突；
5. 先判断当前的key是否与数组上的key是否一致；
6. 当数组上的元素不一致时，则会先判断当前节点是链表还是红黑树，根据不同的策略对key进行判断；
7. 当链表遍历完成后没有找到元素，则会创建新的节点，创建完成后再判断链表的长度是否需要转化为红黑树；转换条件为链表长度大于等于8并且数组容量大于64，否则进行扩容操作。
8. 当元素成功put过后，会使用当前的map容量量与它的阈值进行判断，如果大于则会进行扩容。

### 五、HashMap的put方法源码

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
            // 当table为空时会先对数组进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
            // 根据length-1与hash做与运算得到元素在数组中的位置，如果为空则表示能直接放入，不为空则hash冲突
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
                    // 进行判断是否是map中已有的key
        Node<K,V> e; K k;
                    // 先判断当前节点的hash与key是否相同，相同则代表能直接替换key的value
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
                            // 开始遍历链表
            for (int binCount = 0; ; ++binCount) {
                                    // 当链表下一个元素为空时，则创建新的节点，创建完成后判断是否需要转换为红黑树
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                                    // 链表中有符合的key直接替换旧的key
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
            // 判断是否需要进行扩容操作
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 六、扩容机制

Hashmap在容量超过负载因子所定义的容量之后，就会扩容。方法是将Hashmap的大小扩大为原来数组的两倍，并将原来的对象放入新的数组中。

JDK1.8优化：

> resize之后，元素的位置在原来的位置，或者原来的位置+oldCap(原来哈希表的长度）。不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”。这个设计非常的巧妙，省去了重新计算hash值的时间。

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 原table中已经有值
    if (oldCap > 0) {
        // 已经超过最大限制, 不再扩容, 直接返回
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 注意, 这里扩容是变成原来的两倍
        // 但是有一个条件: `oldCap >= DEFAULT_INITIAL_CAPACITY`
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 在构造函数一节中我们知道
    // 如果没有指定initialCapacity, 则不会给threshold赋值, 该值被初始化为0
    // 如果指定了initialCapacity, 该值被初始化成大于initialCapacity的最小的2的次幂
    // 这里是指, 如果构造时指定了initialCapacity, 则用threshold作为table的实际大小
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 如果构造时没有指定initialCapacity, 则用默认值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 计算指定了initialCapacity情况下的新的 threshold
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    // 从以上操作我们知道, 初始化HashMap时, 
    // 如果构造函数没有指定initialCapacity, 则table大小为16
    // 如果构造函数指定了initialCapacity, 则table大小为threshold, 即大于指定initialCapacity的最小的2的整数次幂
    // 从下面开始, 初始化table或者扩容, 实际上都是通过新建一个table来完成的
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 下面这段就是把原来table里面的值全部搬到新的table里面
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                // 这里注意, table中存放的只是Node的引用, 这里将oldTab[j]=null只是清除旧表的引用, 但是真正的node节点还在, 只是现在由e指向它
                oldTab[j] = null;
                // 如果该存储桶里面只有一个bin, 就直接将它放到新表的目标位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果该存储桶里面存的是红黑树, 则拆分树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 下面这段代码很精妙, 我们单独分一段详细来讲
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
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

它定义了两个链表分别是lo链表和hi链表，loHead和loTail分别指向lo链表的头节点和尾节点，hiHead和hiTail以此类推，按顺序遍历该存储桶位置上的链表中的节点，顺序遍历该存储桶上的链表的每个节点，如果(e.hash&oldCap)==0，我们就将节点放入lo链表，否则放入hi链表.

```java
(e.hash & oldCap) == 0
```

假设oldCap=16,即2^4,16-1=15,二进制表示为00000000000000000000000000001111可见除了低4位,其他位置都是0（简洁起见，高位的0后面就不写了）,则(16-1)&hash自然就是取hash值的低4位,我们假设它为abcd.以此类推,当我们将oldCap扩大两倍后,新的index的位置就变成了(32-1)&hash,其实就是取hash值的低5位.那么对于同一个Node,低5位的值无外乎下面两种情况:

```tex
0abcd
1abcd
```

其中,0abcd与原来的index值一致,而1abcd=0abcd+10000=0abcd+oldCap故虽然数组大小扩大了一倍，但是同一个key在新旧table中对应的index却存在一定联系：要么一致，要么相差一个oldCap。而新旧index是否一致就体现在hash值的第4位(我们把最低为称作第0位),怎么拿到这一位的值呢,只要:hash&00000000000000000000000000010000上式就等效于hash&oldCap。

如果(e.hash&oldCap)==0则该节点在新表的下标位置与旧表一致都为j。

如果(e.hash&oldCap)==1则该节点在新表的下标位置j+oldCap。

总结：

1. resize发生在table初始化,或者table中的节点数超过`threshold`值的时候,`threshold`的值一般为负载因子乘以容量大小.
2. 每次扩容都会新建一个table,新建的table的大小为原大小的2倍.
3. 扩容时,会将原table中的节点re-hash到新的table中,但节点在新旧table中的位置存在一定联系:要么下标相同,要么相差一个oldCap(原table的大小).
4. JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置（头插法）。JDK1.8不会倒置，使用尾插法。

### 七、HashMap线程不安全原因

#### 7.1、JDK1.7

死循环、数据丢失

假设有两个线程同时需要执行resize操作，我们原来的桶数量为2，记录数为3，需要resize桶到4，原来的记录分别为：[3,A],[7,B],[5,C]，在原来的map里面，我们发现这三个entry都落到了第二个桶里面。 假设线程thread1执行到了transfer方法的Entry next = e.next这一句，然后时间片用完了，此时的e = [3,A], next = [7,B]。线程thread2被调度执行并且顺利完成了resize操作，需要注意的是，此时的[7,B]的next为[3,A]。此时线程thread1重新被调度运行，此时的thread1持有的引用是已经被thread2 resize之后的结果。线程thread1首先将[3,A]迁移到新的数组上，然后再处理[7,B]，而[7,B]被链接到了[3,A]的后面，处理完[7,B]之后，就需要处理[7,B]的next了啊，而通过thread2的resize之后，[7,B]的next变为了[3,A]，此时，[3,A]和[7,B]形成了环形链表，在get的时候，如果get的key的桶索引和[3,A]和[7,B]一样，那么就会陷入死循环。

```java
/**
* Transfers all entries from current table to newTable.
* 将所有的entry数组从现在的table移到newTable
*/
 void transfer(Entry[] newTable, boolean rehash) {  
    int newCapacity = newTable.length;  
    for (Entry<K,V> e : table) {  
        while(true) {  
            Entry<K,V> next = e.next;  
            // 如果hash冲突了
            if (rehash) {  
                e.hash = null == e.key ? 0 : hash(e.key);  
            }
            // 重新求新的数组下表
            int i = indexFor(e.hash, newCapacity);
            // e的下一个就是下标为i的newTable数组元素
            e.next = newTable[i];  
            newTable[i] = e;  
            e = next;  
        } 
    }  
}
```

#### 7.2、JDK1.8

数据覆盖

第六行代码是判断是否出现hash碰撞，假设两个线程A、B都在进行put操作，并且hash函数计算出的插入下标是相同的，当线程A执行完第六行代码后由于时间片耗尽导致被挂起，而线程B得到时间片后在该下标处插入了元素，完成了正常的插入，然后线程A获得时间片，由于之前已经进行了hash碰撞的判断，所有此时不会再进行判断，而是直接进行插入，这就导致了线程B插入的数据被线程A覆盖了，从而线程不安全。这里要注意的一个点就是，这个的覆盖不安全是由于是不同key所引起的，不是正常的HashMap自带的由于相同的key引起的覆盖，因此这里是不安全的。

除此之前，还有就是代码的第38行处有个++size，我们这样想，还是线程A、B，这两个线程同时进行put操作时，假设当前HashMap的zise大小为10，当线程A执行到第38行代码时，从主内存中获得size的值为10后准备进行+1操作，但是由于时间片耗尽只好让出CPU，线程B快乐的拿到CPU还是从主内存中拿到size的值10进行+1操作，完成了put操作并将size=11写回主内存，然后线程A再次拿到CPU并继续执行(此时size的值仍为10)，当执行完put操作后，还是将size=11写回内存，此时，线程A、B都执行了一次put操作，但是size的值只增加了1，所有说还是由于数据覆盖又导致了线程不安全。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

