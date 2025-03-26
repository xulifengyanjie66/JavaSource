# HashMap 底层源码分析

## 目录
- [1. 引言](#1-引言)
- [2. HashMap 的数据结构](#2-hashmap-的数据结构)
- [3. HashMap 的核心参数](#3-hashmap-的核心参数)
- [4. HashMap 的核心方法](#4-hashmap的核心方法)
   - [4.1 hash() 方法解析](#41-hash-方法解析)
   - [4.2 put() 方法解析](#42-put-方法解析)
   - [4.3 get() 方法解析](#43-get-方法解析)
   - [4.4 resize() 扩容机制](#44-resize-扩容机制)
- [5. HashMap 的线程安全问题](#5-hashmap的线程安全问题)
- [6. HashMap 的 JDK 版本演变](#6-hashmap-的-jdk-版本演变)
- [7. 结论](#7-结论)



## 1. 引言
`HashMap` 是 Java **最常用** 的集合之一，底层基于 **数组 + 链表 + 红黑树** 进行存储。本篇文章将分析 `HashMap` **源码**。


## 2. HashMap 的数据结构
在 JDK 1.8 之前，`HashMap` 采用 **数组 + 链表** 结构，而JDK1.8之后，引入了 **红黑树** 来优化性能：
- **数组（Node<K, V>[] table）**：存储键值对的主要结构。
- **链表**：用于解决 **哈希碰撞**。
- **红黑树**：当链表长度超过 `8` 时，会转换为 **红黑树** 以提高查找效率。



## 3. HashMap 的核心参数
在 `HashMap` 中，常见的几个核心参数：
- **DEFAULT_INITIAL_CAPACITY = 16**（默认初始容量）
- **MAXIMUM_CAPACITY = 1 << 30**（最大容量）
- **DEFAULT_LOAD_FACTOR = 0.75f**（默认负载因子）
- **TREEIFY_THRESHOLD = 8**（链表转换为红黑树的阈值）
- **UNTREEIFY_THRESHOLD = 6**（红黑树退化为链表的阈值）



## 4. HashMap的核心方法
### 4.1 hash() 方法解析
&nbsp;&nbsp;本次源码分析基于JDK21,在分析put源码之前我们先看一个hash方法，它是被put方法调用的，比较重要，它是计算key的hash值的然后存放在数组相应位置上，
它的方法签名是这样的。
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
这段获取hashCode代码为啥要这样写?hash()函数通过key.hashCode() ^ (h >>> 16)计算哈希值，‌核心目的是优化哈希值的分布，减少哈希冲突‌。具体原因和实现逻辑如下：

- 背景：哈希冲突与低位碰撞

  &nbsp;&nbsp;哈希表通常用取模运算（如index = hash % tableSize）定位桶的位置。当哈希表的容量较小时（例如默认容量16），‌取模运算仅依赖哈希值的低位‌：
  如果对象的hashCode()主要集中在高位变化，而低位重复度高，会导致大量哈希冲突。
  例如有以下代码：

  &nbsp;&nbsp;//假设两个对象的哈希码分别为：

  &nbsp;&nbsp;Object1.hashCode() = 0x12345678

  &nbsp;&nbsp;Object2.hashCode() = 0xABCD5678

  &nbsp;&nbsp;//哈希表容量为16时，取模运算用低4位：

  &nbsp;&nbsp;0x12345678 % 16 = 0x8 → 桶8

  &nbsp;&nbsp;0xABCD5678 % 16 = 0x8 → 桶8 → 冲突！

  &nbsp;&nbsp;尽管高位不同，但低位相同会导致冲突。
- 解决方案: 混合高位与低位信息 通过 ‌hashCode() ^ (hashCode() >>> 16)‌ 将哈希码的高16位与低16位混合

  &nbsp;&nbsp;高位右移‌：h >>> 16 将高16位移动到低16位。 ‌异或（XOR）‌：将原始低16位与高位信息混合，生成新的低16位。 操作步骤： ‌计算原始哈希码‌：h = key.hashCode()（32位）。 ‌右移16位‌：h >>> 16 → 高16位移动到低16位，高16位补零。 ‌异或运算‌：h ^ (h >>> 16) → 将高位信息“扩散”到低位。

  示例:
  
  h = 0x12345678 h >>> 16 = 0x00001234

  异或后： 0x12345678 ^ 0x00001234 = 0x1234444C

  取模运算时，低16位是0x444C（不再是重复的0x5678）→ 减少冲突。
- 为什么用异或（XOR）而不是其他位运算？

  &nbsp;&nbsp;异或的特性:‌异或能保证高低位信息‌均匀混合‌，避免偏向0或1(与运算偏向0，或运算偏向1)。
  ‌效率:‌异或是单周期CPU指令，性能最优。

### 4.2 put() 方法解析
&nbsp;&nbsp;put方法调用了上面说的hash方法，接着调用了putVal方法，该方法的签名如下：
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
&nbsp;&nbsp;定义四个变量分别如下：Node<K,V>[] tab; Node<K,V> p; int n, i 把table赋值给临时变量tab，判断是否为空，首次指定为null,调用resize方法，把tab的长度赋值给一个临时变量oldCap,判断它是否大于0，那我们先
分析首次肯定是不大于0的，所以newCap=DEFAULT_INITIAL_CAPACITY，DEFAULT_INITIAL_CAPACITY是多少那，它是 1 << 4,即16,newThr=0.75 * 16。

&nbsp;&nbsp;把它赋值给成员变量threshold表示超过这个值就需要扩容，创建一个容量是16的Node数组，然后赋值给成员变量table,此时调用返回到putVal方法n=16,
此时计算 (n-1) & hash计算出下标值i,判断tab[i]是否等于null,首次添加指定是null,那就新创建一个对象Node,把hash值赋值给成员变量hash,key,value同样赋值,next指向null。
然后tab[i] =Node对象，修改次数成员变量modCount加1,size加1判断是否大于上面提到的threshold,如果大于就调用扩容方法，这里先暂且不分析扩容逻辑，就这样首次调用put方法逻辑
就分析完成了。

&nbsp;&nbsp;接着分析第二次添加的时候，假如此时发生了冲突，即p = tab[i = (n - 1) & hash]) != null,此时p的值是发生冲突时之前已经存在的值，此时分三种情况判断，
判断要添加的hash是否等于冲突的hash值并且key是否相等，如果相等，说明两个对象是同一个对象，把p的值赋值给临时变量e,如果e不为空，先保留老的oldValue值,
把之前的value值赋值为要添加的value,即覆盖已经存在的value值，返回老的value值。

&nbsp;&nbsp;如果判断的hash值和要添加的hash值不相等或者key不相等，判断p是否是TreeNode对象，即红黑树对象，此次逻辑我咱且不分析。

&nbsp;&nbsp;如果也不是红黑树对象，走到else语句，这里首先是一个死循环for,for循环里面也有两个逻辑，如果冲突元素的next节点不等于null,
此时e=p.next,判断e元素的hash值是否等于要添加的元素hash,e元素的key是否等于要添加的元素key,如果相等跳出for循环，后续还是把链表的值替换为
要添加的value值，如果hash或者key不相等把p=e,此时又判断p.next是否等于空，如果为空，说明已经遍历到链表的结尾，此时p.next=新创建元素，此处有个判断逻辑是
if (binCount >= TREEIFY_THRESHOLD - 1)
treeifyBin(tab, hash);
说明链表长度大于8了调用treeifyBin方法，这个方法又判断数组的长度是否大于64，如果大于转换为红黑树，如果小于就扩容，关于红黑树的逻辑这里还是先不介绍了，感兴趣的可以自己去看看。
到此为止put的源码就分析完成了。

### 4.3 get() 方法解析
&nbsp;&nbsp;该方法的签名如下：
```java
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(key)) == null ? null : e.value;
}

final Node<K,V> getNode(Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n, hash; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
    (first = tab[(n - 1) & (hash = hash(key))]) != null) {
      if (first.hash == hash && // always check first node
      ((k = first.key) == key || (key != null && key.equals(k))))
      return first;
    if ((e = first.next) != null) {
      if (first instanceof TreeNode)
      return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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
&nbsp;&nbsp;定义一个临时变量Node e,调用getNode方法传入要查找的key,getNode方法中定义临时变量tab,first,e,n,hash,k,把table赋值给变量tab判断
是否不等于null,把tab的长度赋值给变量n,判断是否大于0,计算key的hash值在table处的索引值，并且判断该处元素是否为空null,如果有一个条件不满足就返回null值，说明没找到该key对应的value值。

&nbsp;&nbsp;如果条件成立first变量就是要找的元素，判断first的hash值与key计算的hash值是否相等并且first的key是否等于传入的key或者比较两者的equals值是否相等，如果都相等返回first,然后拿到里面的value属性返回。
如果不相等说明发生了hash冲突,把first的next赋值给变量e,判断first是否是红黑树，如果是调用getTreeNode获取，如果不是说明是链表结构，进入一个do while循环，判断e的hash值是否等于key的hash值并且
e的key是否等于key或者比较两者的equals值是否相等，如果不相等就一直获取当前元素的next节点直到相等为止返回元素的value值。

### 4.4 resize() 扩容机制
&nbsp;&nbsp;resize方法的签名如下:
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
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
把table数组赋值给oldTab变量，判断oldTab是否等于null,此处我们假设不为空，即原始数组已经装不下需要扩容的情况，此时oldCap等于数组的长度，oldThr等于原来的threshold，定义newCap, newThr等于0,
如果oldCap大于0并且大于等于MAXIMUM_CAPACITY(2的30次方),threshold等于Integer的最大值,返回oldTab表示不能再进行扩容。

否则把oldCap向左移动1，得到新的newCap容量是32，然后得到新的扩容阈值newThr=24，把newThr赋值给成员变量threshold,新创建一个数组newTab,容量是32，成员变量table
指向newTab。

接下来循环遍历老的数组，重新计算它们的索引下标，把它们重新放到新数组的相应位置，这里分几种情况，一是遍历元素没有冲突，即没有链表和红黑树的情况下;二是红黑树的情况下;三是链表的情况下，下面我只分析没有冲突和有冲突是链表结构的情况，红黑树的情况暂且不分析。
- 没有冲突的resize逻辑

&nbsp;&nbsp;循环遍历把对应下标的元素赋值给临时变量Node e,并把当前的对应下标元素指向null,用原来元素的hash值和新数组容量(32)减去1进行&操作重新计算下标值用于存放原来的元素。

- 链表的resize逻辑

&nbsp;&nbsp;这块的逻辑主要是处理链表桶（非红黑树）的情况,它的主要逻辑是对旧桶中的链表进行重新分配，将节点拆分到新的newTab数组中不同的桶位置。
loHead / loTail：低位链表的头尾指针,hiHead / hiTail：高位链表的头尾指针,这段代码的目的就是将旧链表拆分成两个部分：一个是低位链表(index = j)：e.hash & oldCap == 0;
一个是高位链表(index = j + oldCap)：e.hash & oldCap != 0,下面一步步进行分析。

&nbsp;&nbsp;如果e.hash & oldCap) == 0说明是低位，判断loTail是否等于null,假如第一次执行时候指定是null,loHead=e,loTail=e,第二次执行时候loTail不是null,则loTail.next=e,loTail指向当前e,这里举个例子说明
假设 oldCap = 16,newCap = 32,并且 table[3] 这个桶中有以下链表：3 -> 19 -> 35 -> 7,假设它们的 hash 分别为：
hash(3)  = 0011  -> (hash & oldCap) == 0  -> 低位;hash(19) = 10011 -> (hash & oldCap) != 0 -> 高位; hash(35) = 100011 -> (hash & oldCap) != 0 -> 高位;hash(7)  = 0111  -> (hash & oldCap) == 0  -> 低位,那么执行上面低位逻辑以后
loHead 低位链表 = 3 -> 7,loTail=7,do while执行以后判断loTail不等于null, loTail.next = null,断开原链表，避免循环, newTab[j] = loHead,然后放入新的数组桶中。

&nbsp;&nbsp;同理高位的处理逻辑类似，只不过放入新数组下标位置是当前遍历的下标加上oldCap，目的是放入新数组的高位，这样做能有效降低hash冲突的概率。


## 5. HashMap的线程安全问题

`HashMap` **在多线程环境下** 存在以下问题：
- **JDK 1.7：并发 put() 可能导致死循环（形成环形链表）**
- **JDK 1.8：改用尾插法，但仍然不是线程安全的**
- **并发下的常见问题**：
   - 扩容时可能导致 **数据丢失**
   - 可能出现 **无限循环**
   - 可能导致 **数据覆盖**
  
### **5.1 线程安全解决方案**
| 方案 | 说明 |
|------|-----------------------------------|
| `Collections.synchronizedMap(new HashMap<>())` | 适用于少量并发，使用 `synchronized`，性能较差 |
| `ConcurrentHashMap` | 适用于高并发，JDK 8+ 采用 CAS + 自旋锁 |
| `Hashtable` | 线程安全，但使用 `synchronized` 进行全表锁，效率较低 |
**推荐** 在并发场景下使用 `ConcurrentHashMap`，而不是 `HashMap`。

---

## 6. HashMap 的 JDK 版本演变
在不同 JDK 版本中，`HashMap` 进行了优化和调整：

| 版本 | 数据结构 | 主要特点 |
|------|--------------------|----------------------------|
| **JDK 1.7** | 数组 + 链表 | 头插法，容易形成环形链表 |
| **JDK 1.8** | 数组 + 链表 + 红黑树 | 尾插法，链表超过 8 转红黑树 |
| **JDK 11+** | 数组 + 链表 + 红黑树 | 进一步优化哈希分布，提高性能 |

JDK 1.8 **最大的优化点** 是：
- 采用 **尾插法** 避免死循环问题
- **链表长度 > 8 时，自动转换为红黑树**
- **扩容时避免 rehash，提高效率**

---

## 7. 结论
- `HashMap` 采用 **数组 + 链表 + 红黑树** 进行存储，默认初始容量为 `16`，负载因子 `0.75`，扩容为 `2` 倍。
- **JDK 1.8 之后** 采用 **尾插法** 解决了 **死循环问题**，并引入 **红黑树** 提升查询效率。
- **HashMap 不是线程安全的**，并发场景下应使用 `ConcurrentHashMap` 代替。
- 了解 `HashMap` 的底层原理有助于编写 **高效、健壮** 的代码，避免 **错误使用** 造成的性能问题。