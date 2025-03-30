# LinkedHashMap源码分析及LRU实现原理

## 目录
- [1. 引言](#1-引言)
- [2. LinkedHashMap 概述](#2-linkedhashmap-概述)
- [3. 继承关系分析](#3-继承关系分析)
   - [3.1 类继承结构](#31-类继承结构)
- [4. LinkedHashMap 的数据结构](#4-linkedhashmap-的数据结构)
- [5. LinkedHashMap 的访问顺序与插入顺序](#5-linkedhashmap-的访问顺序与插入顺序)
- [6. LRU 缓存的实现原理](#6-lru-缓存的实现原理)
    - [6.1 removeEldestEntry 方法](#6-lru-缓存的实现原理)
- [7. 源码解析](#7-源码解析)
    - [7.1 newNode 方法](#71-newnode方法)
    - [7.2 afterNodeAccess 方法](#72-afternodeaccess方法)
    - [7.3 afterNodeInsertion 方法](#73-afternodeinsertion方法lru缓存应用)
    - [7.4 get 方法](#74-get方法)
    - [7.5 forEach 方法](#75-foreach方法)
- [8. 总结](#8-总结)

---

## 1. 引言
在 Java 集合框架中，`LinkedHashMap` 既具有 `HashMap` 的高效查找能力，又能维护元素的插入顺序或访问顺序。本文将深入分析 `LinkedHashMap` 的源码，并探讨其实现 LRU（Least Recently Used）缓存的原理。

## 2. LinkedHashMap 概述
`LinkedHashMap` 继承自 `HashMap`，但它额外维护了一个 **双向链表**，用于维护元素的顺序。其主要特点包括：
- 可以按照 **插入顺序** 或 **访问顺序** 迭代元素。
- 提供 `removeEldestEntry` 方法，方便实现 LRU 缓存。

---

## 3. 继承关系分析
### 3.1 类继承结构
`LinkedHashMap` 继承自 `HashMap`，并间接实现了 `Map` 接口：

```plaintext
java.lang.Object
 └── java.util.AbstractMap<K,V>
      └── java.util.HashMap<K,V>
           └── java.util.LinkedHashMap<K,V>
```           
### 3.2 类的继承结构图
![img8.jpg](..%2Fimg%2Fimg8.jpg)

&nbsp;&nbsp;从类的继承结构图可以看出LinkedHashMap 实现了 Map<K,V> 和 Cloneable，同时支持序列化：

- 继承 HashMap：继承了 put()、get() 等核心方法，并通过 afterNodeAccess() 等方法增强功能。
- 实现 Map<K,V>：保证其符合 Map 接口规范。
- 实现 Cloneable：支持对象克隆。
- 实现 Serializable：可序列化，支持对象存储和传输。

## 4. LinkedHashMap 的数据结构

LinkedHashMap 采用 双向链表 + HashMap 结构，其中：
- HashMap 用于快速查找。
- 双向链表 用于维护迭代顺序。

它的 Entry<K,V> 继承自 HashMap.Node<K,V>，并增加了 before 和 after 指针，形成 双向链表。

## 5. LinkedHashMap 的访问顺序与插入顺序

&nbsp;&nbsp;LinkedHashMap 提供了 两种顺序模式：

- 插入顺序（默认）：元素按照插入的先后顺序进行迭代。
- 访问顺序：当 accessOrder = true 时，每次访问一个元素，该元素会被移动到链表末尾。

## 6. LRU 缓存的实现原理
&nbsp;&nbsp;LinkedHashMap 通过 removeEldestEntry 方法自动移除最久未使用的元素，配合 访问顺序 实现 LRU 机制：
```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > MAX_CAPACITY;
}
```
当 size() 超过 MAX_CAPACITY 时，最早的元素（链表头部）会被移除,关于此处我将在源码部分进行详细分析。

## 7. 源码解析
### 7.1 newNode方法
&nbsp;&nbsp;LinkedHashMap重写了HashMap的newNode方法，所以有必要看看这个代码的实现，该方法的签名如下:
```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<>(hash, key, value, e);
    linkNodeAtEnd(p);
    return p;
}
```
由于put方法是继承了HashMap的，上篇文章已经分析过了，这里就不再重复分析了，我们重点看这个方法的逻辑，先是创建了LinkedHashMap.Entry的对象，同样是传入hash值，key,value和next是null的值,
然后调用了linkNodeAtEnd方法传入LinkedHashMap.Entry对象，该方法的签名如下：
```java
private void linkNodeAtEnd(LinkedHashMap.Entry<K,V> p) {
      if (putMode == PUT_FIRST) {
          LinkedHashMap.Entry<K,V> first = head;
          head = p;
          if (first == null)
              tail = p;
          else {
              p.after = first;
              first.before = p;
          }
      } else {
          LinkedHashMap.Entry<K,V> last = tail;
          tail = p;
          if (last == null)
              head = p;
          else {
              p.before = last;
              last.after = p;
          }
      }
}
```
判断putMode是否等于PUT_FIRST,默认是不等于的，这里会走到else语句,第一次put时候last=tail=null,此时tail指向p,此时head也指向p,假如第一次添加的是3
那么此时tail=head=3,第二次添加的时候比如是4,执行到else语句,此时last=3,tail=4,4的before节点是3,3的last节点是4，这时就形成了一个双向链表结构。

### 7.2 afterNodeAccess方法
&nbsp;&nbsp;LinkedHashMap的afterNodeAccess方法,用于在访问（或插入）节点后调整双向链表的顺序，以实现LRU（最近最少使用）或自定义插入顺序的特性,该方法调用是在put方法发生冲突时候或者get方法且是LRU(accessOrder=true)时才调用，
该方法的源码如下：
```java
void afterNodeAccess(Node<K,V> e) {
        LinkedHashMap.Entry<K,V> last;
        LinkedHashMap.Entry<K,V> first;
        if ((putMode == PUT_LAST || (putMode == PUT_NORM && accessOrder)) && (last = tail) != e) {
            // move node to last
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        } else if (putMode == PUT_FIRST && (first = head) != e) {
            // move node to first
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.before = null;
            if (a == null)
                tail = b;
            else
                a.before = b;
            if (b != null)
                b.after = a;
            else
                first = a;
            if (first == null)
                tail = p;
            else {
                p.after = first;
                first.before = p;
            }
            head = p;
            ++modCount;
        }
}
```
判断accessOrder是否等于true，如果是说明是LRU访问模式,每次访问节点（如 get()）会将节点移到链表尾部（表示最近使用）。accessOrder=false（插入顺序模式），链表保持插入顺序,
通过 putMode 控制插入新节点时放在链表头部（PUT_FIRST）还是尾部（PUT_LAST），下面一步步分析其源码。

如果是LRU或者PUT_LAST模式并且当前节点不是尾部节点,此时假如有这样的代码，我以这个代码举例说明:
```java
LinkedHashMap linkedHashMap = new LinkedHashMap(1,0.75f,true);
linkedHashMap.put("33","33");
linkedHashMap.put("55","33");
linkedHashMap.put("33","44");
```
当执行到linkedHashMap.put("33","44")时候就会走到LRU的判断中,此时p是(33,44),b=p.before,此时b是null,a=p.after,此时a是(55,33),把p.after指向null,意味着要把p移动到链表尾部，
判断如果是b=null,说明当前要移动元素是头节点，此时更新头节点是p.after即a(55,33)为头节点，如果移动的不是头节点，将前驱节点b指向后继节点a,
如果a不等于null,将后继节点a指向前驱节点b,如果p是原尾节点，更新尾指针，如果链表为空时，p成为头节点，否则p.before指向last节点，last.after = p,最后把tail尾部节点指向
p,经过这一系列操作，最终链表的结构是(55,33)->(33,44),head指向(55,33),tail指向(33,44),可以看出如果是LRU或者PUT_LAST模式，在冲突或者访问时候会把元素排列到后面。

### 7.3 afterNodeInsertion方法(LRU缓存应用)
&nbsp;&nbsp;LinkedHashMap.afterNodeInsertion(boolean evict) 是HashMap在插入新节点后调用的一个回调方法，由LinkedHashMap 重写，用于实现自动移除最旧节点的功能（常用于实现 LRU 缓存）。它的核心作用是：
在插入新节点后，检查是否需要移除链表头部的节点（最久未使用的节点），以控制 Map 的大小,该方法的签名如下：
```java
void afterNodeInsertion(boolean evict) {
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
}

void afterNodeRemoval(Node<K,V> e) {
   LinkedHashMap.Entry<K,V> p =
    (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
   p.before = p.after = null;
   if (b == null)
     head = a;
   else
     b.after = a;
   if (a == null)
      tail = b;
   else
     a.before = b;
}
```
**参数说明**

- evict：是否允许移除旧节点（true 表示允许，false 表示不允许）。
- first：链表头节点（即最旧的节点）。

**执行逻辑**

1.**检查是否允许移除节点（evict 为 true）。**

2.**检查链表是否非空（head != null）。**

3.**检查是否需要移除最旧节点（removeEldestEntry(first) 返回 true）。**

4.**如果满足条件，移除头节点（调用 removeNode 删除该节点）。**

LinkedHashMap提供了一个可重写的方法removeEldestEntry默认返回false,作用是决定是否移除链表头节点（最旧的节点），典型的应用是LRU缓存。

下面我会写一段代码来举例说明LRU缓存的应用，请看下面的代码:
```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;

    public LRUCache(int maxSize) {
        super(maxSize, 0.75f, true); // accessOrder=true（LRU 模式）
        this.maxSize = maxSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize; // 超过容量时移除最旧节点
    }
}

LRUCache<String, Integer> cache = new LRUCache<>(3);
cache.put("A", 1);
cache.put("B", 2);
cache.put("C", 3);
cache.put("D", 4); // 触发移除 "A"
System.out.println(cache); // 输出：{B=2, C=3, D=4}
```
我这里自定义一个类继承LinkedHashMap,并重写了removeEldestEntry方法，该方法表示该map的size超出最大容量时移除链表头部元素，下面我就以此代码分析
afterNodeInsertion的执行流程。

当cache.put("D",4)时候，由于此时size已经时4了大于maxSize=3,此时removeEldestEntry方法会返回true,此时会执行afterNodeInsertion方法的if语句块的逻辑
传入链表的头部节点，获取key值，接着调用hash方法获取key对应的hash值，又调用了removeNode方法，最终调用到afterNodeRemoval方法，这块我们只考虑删除头节点逻辑，因为
LRU就是删除头节点，把p的before和after执行都指向null,以便GC能够回收该节点，把p.after变为头节点。

### 7.4 get方法
&nbsp;&nbsp;该方法的签名如下:
```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```
从源码可以看出主要逻辑还是调用了父类的getNode方法，如果是LRU模式，会调用afterNodeAccess方法把访问的元素放到链表尾部。

### 7.5 forEach方法

&nbsp;&nbsp;为什么要分析forEach方法那，因为从这个方法里面就可以看出为啥LinkedHashMap可以按照插入元素顺序输出结果,下面是这个方法的签名：
```java
public void forEach(BiConsumer<? super K, ? super V> action) {
      if (action == null)
          throw new NullPointerException();
      int mc = modCount;
      for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
          action.accept(e.key, e.value);
      if (modCount != mc)
          throw new ConcurrentModificationException();
  }
```
从这个方法可以看出，它是遍历双向循环链表，从头到尾，那如果是正常插入模式，先添加的指定在前面，后添加的就在后面，这就可以解释为什么LinkedHashMap可以按照插入元素顺序输出结果。

## 8. 总结

- LinkedHashMap继承自HashMap，但额外维护了双向链表，支持按插入顺序或访问顺序迭代元素。

- 通过 accessOrder = true 和 removeEldestEntry，可以轻松实现 LRU 缓存。

- afterNodeAccess、afterNodeInsertion和afterNodeRemoval 方法负责维护双向链表，确保访问顺序正确。

通过本次源码分析，我们可以更深入地理解 LinkedHashMap的实现细节及其在LRU 缓存中的应用。
