# TreeMap源码分析与红黑树实现原理

## 目录
- [1. 引言](#1-引言)
- [2. 继承关系分析](#2-继承关系分析)
- [3. TreeMap的数据结构](#3-treemap的数据结构)
   - [3.1 TreeMap的核心成员变量](#31-treemap的核心成员变量)
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
&nbsp;&nbsp;TreeMap是Java集合框架中的一个重要成员，基于红黑树（Red-Black Tree）实现，能够保证键值对按照键的顺序排列。本文将深入分析TreeMap 的源码，本文采用的JDK版本是21,重点介绍其数据结构、核心操作以及红黑树的实现原理。

## 2. 继承关系分析

&nbsp;&nbsp;TreeMap 继承自 AbstractMap<K, V>，并实现了 NavigableMap<K, V> 和 SortedMap<K, V> 接口。

该类的继承结构图如下图所示：

![img9.jpg](..%2Fimg%2Fimg9.jpg)

## 3. TreeMap的数据结构
### 3.1 TreeMap的核心成员变量
```java
private transient Entry<K,V> root;  // 根节点
private transient int size = 0;      // TreeMap 的大小
private final Comparator<? super K> comparator; // 比较器（可选）
```
### 3.2 Entry<K,V>节点结构
```java
static final class Entry<K,V> implements Map.Entry<K,V>{
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = RED; // 新插入的节点默认是红色
}
```
&nbsp;&nbsp;其中:

- key/value：键值对
- left/right：左右子节点
- parent：父节点
- color：节点颜色（红/黑）
## 4. 红黑树的基本概念
&nbsp;&nbsp;红黑树是一种 自平衡二叉搜索树（BST），具备以下性质：

- 每个节点非红即黑。

- 根节点始终是黑色。

- 红色节点的子节点必须是黑色（即不能出现两个连续的红色节点）。

- 从任一节点到叶子节点的所有路径，黑色节点数必须相同（黑色平衡性）。

- 新插入的节点默认是红色（避免破坏黑色平衡）。

### 4.1 红黑树的操作
&nbsp;&nbsp;红黑树的三种操作：变色、左旋、右旋。
- **变色:**
  
  节点的颜色由红变黑或由黑变红。(这个操作很好了解)
- **左旋:**
  
  以某个结点作为支点(pivot),其父节点（子树的root）旋转为自己的左子树（左旋），pivot的原左子树变成原root节点的右子树，pivot的原右子树保持不变,我这里有张图方便理解

  ![img10.png](..%2Fimg%2Fimg10.png)


  &nbsp;&nbsp;&nbsp;&nbsp;从这张图可以看出,以18这个元素为pivot,把其父节点9旋转为自己的左子树,pivot的原左子树10变成root节点9的右子树

- **右旋:**

  以某个结点作为支点(pivot),其父节点（子树的root）旋转为自己的右子树（右旋），pivot的原右子树变成 原root节点的左子树，pivot的原左子树保持不变,我这里有张图方便理解

  ![img11.png](..%2Fimg%2Fimg11.png)


  &nbsp;&nbsp;&nbsp;&nbsp;从这张图可以看出,以7这个元素为pivot,把其父节点9旋转为自己的右子树,pivot的原右子树8变成root节点9的左子树

### 4.2 红黑树插入场景旋转分析
&nbsp;&nbsp;实际应用插入的场景中，红黑树的旋转情况非常多，下面我一一例举几种场景，为源码分析做准备

- **场景1:** 红黑树为空树

直接把插入结点作为根节点就可以了并且把插入节点设置为黑色。

- **场景2:** 插入节点的父节点为黑色
由于插入的节点是红色的，当插入节点的父节点是黑色时，不会影响红黑树的平衡，如下图所示:

 ![img12.png](..%2Fimg%2Fimg12.png)

- **场景3:** 插入节点的父节点为红色
  
&nbsp;&nbsp;如果插入节点的父节点为红色节点,由于新插入得节点也为红色，根据规则不能有两个相邻得红色节点,此时分两种情况考虑
- **父亲和叔叔均为红色**
- **父亲为红色，叔叔为黑色**

如图(K为要插入的元素):

![img13.jpg](..%2Fimg%2Fimg13.jpg)

**场景3.1：父亲和叔叔为红色节点**

父亲为红色，那么此时该插入子树的红黑树层数的情况是：黑红红。

因为不可能同时存在两个相连的红色节点,需要进行变色,显然处理方式是把其改为：红黑红。

**变色处理：** 黑红红 ==> 红黑红

1.将F和V节点改为黑色

2.将P改为红色

3.将P设置为当前节点，进行后续处理

此时操作如下图：

![img14.png](..%2Fimg%2Fimg14.png)

**场景3.2：叔叔为黑色，父亲为红色，并且插在父亲的左节点**

&nbsp;&nbsp;分为两种情况

- LL红色插入

叔叔为黑色，或者不存在(NIL)也是黑节点，并且节点的父亲节点是祖父节点的左子节点

如图:

![img15.png](..%2Fimg%2Fimg15.png)

**场景3.2.1 LL型失衡**

&nbsp;&nbsp;细分场景 1： 新插入节点，为其父节点的左子节点(LL红色情况),插入后就是LL型失衡,如图所示：

![img16.png](..%2Fimg%2Fimg16.png)

**自平衡处理：**

1.变颜色：

将F设置为黑色，将P设置为红色

2.对F节点进行右旋

如图:

![img17.png](..%2Fimg%2Fimg17.png)

**场景3.2.2 LR型失衡**

细分场景2:新插入节点，为其父节点的右子节点(LR红色情况),插入后就是LR型失衡,如图所示:

![img18.png](..%2Fimg%2Fimg18.png)

**自平衡处理：**

1.对F进行左旋

2.将F设置为当前节点，得到LL红色情况

3.按照LL红色情况处理(1.变色 2.右旋P节点)

![img19.png](..%2Fimg%2Fimg19.png)

**场景3.3：叔叔为黑节点，父亲为红色，并且父亲节点是祖父节点的右子节点**

![img20.png](..%2Fimg%2Fimg20.png)

**场景3.3.1：RR型失衡**

&nbsp;&nbsp;新插入节点，为其父节点的右子节点(RR红色情况)

![img22.png](..%2Fimg%2Fimg22.png)

**自平衡处理：**

1.变色：

将F设置为黑色，将P设置为红色

2.对P节点进行左旋

![img23.png](..%2Fimg%2Fimg23.png)

**场景3.3.2：RL型失衡**

&nbsp;&nbsp;新插入节点，为其父节点的左子节点(RL红色情况)

![img24.png](..%2Fimg%2Fimg24.png)

**自平衡处理：**

1.对F进行右旋

2.将F设置为当前节点，得到RR红色情况

3.按照RR红色情况处理(1.变色 2.左旋 P节点)

![img25.png](..%2Fimg%2Fimg25.png)

## 5. TreeMap源码分析

### 5.1 插入(put方法)

&nbsp;&nbsp;该方法的签名如下：

```java
private V put(K key, V value, boolean replaceOld) {
        Entry<K,V> t = root;
        if (t == null) {
            addEntryToEmptyMap(key, value);
            return null;
        }
        int cmp;
        Entry<K,V> parent;

        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else {
                    V oldValue = t.value;
                    if (replaceOld || oldValue == null) {
                        t.value = value;
                    }
                    return oldValue;
                }
            } while (t != null);
        } else {
            Objects.requireNonNull(key);
            Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else {
                    V oldValue = t.value;
                    if (replaceOld || oldValue == null) {
                        t.value = value;
                    }
                    return oldValue;
                }
            } while (t != null);
        }
        addEntry(key, value, parent, cmp < 0);
        return null;
}
```
首先分析第一次添加的时候,比如添加的是元素3,把root成员属性赋值给变量t,第一次t是null,调用addEntryToEmptyMap方法，该方法的签名如下:
```java
private void addEntryToEmptyMap(K key, V value) {
        compare(key, key); 
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
}
```
主要是创建一个Entry类型的root对象,设置size等于1,modCount加1，最终返回null,首次添加的逻辑很简单。

接着分析第二次添加的时候，比如添加的是元素2,定义int类型变量cmp,Entry类型的parent,把成员变量类型是Comparator的comparator赋值给cpr,
判断它是否为null,这块的comparator可以由调用端自行传入，假如没传递这个，会走到else语句中,把key强制转换为Comparable类型，从这块可以看出，如果是
自定义类型的key，需要实现Comparable接口，否则会报ClassCastException异常，相信这里大家都理解,这时候有do while循环,把元素t也是root赋值给变量parent, 用当前插入元素和根元素比较，
如果小于根元素说明是根元素的左子树，把根元素的左子树赋值给变量t,判断t是否为空，如果不为空继续判断直到null为止，经过这一系列do while循环比较就能找出
新传入的元素父亲节点parent,如果比较以后相等说明就是要覆盖之前的老值，然后返回老值。

找到要插入元素的父亲节点以后，调用addEntry方法，该方法的签名如下；
```java
  private void addEntry(K key, V value, Entry<K, V> parent, boolean addToLeft) {
      Entry<K,V> e = new Entry<>(key, value, parent);
      if (addToLeft)
          parent.left = e;
      else
          parent.right = e;
      fixAfterInsertion(e);
      size++;
      modCount++;
  }
```
创建Entry对象，传入key、value、父亲节点，如果要插入到左子树把当前插入的元素赋值给parent.left,如果是右子树把当前插入的元素赋值给parent.right，插入以后需要进行自平衡操作，调用方法
fixAfterInsertion,该方法的签名如下:
```java
private void fixAfterInsertion(Entry<K,V> x) {
        x.color = RED;

        while (x != null && x != root && x.parent.color == RED) {
            if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
                Entry<K,V> y = rightOf(parentOf(parentOf(x)));
                if (colorOf(y) == RED) {
                    setColor(parentOf(x), BLACK);
                    setColor(y, BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    x = parentOf(parentOf(x));
                } else {
                    if (x == rightOf(parentOf(x))) {
                        x = parentOf(x);
                        rotateLeft(x);
                    }
                    setColor(parentOf(x), BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    rotateRight(parentOf(parentOf(x)));
                }
            } else {
                Entry<K,V> y = leftOf(parentOf(parentOf(x)));
                if (colorOf(y) == RED) {
                    setColor(parentOf(x), BLACK);
                    setColor(y, BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    x = parentOf(parentOf(x));
                } else {
                    if (x == leftOf(parentOf(x))) {
                        x = parentOf(x);
                        rotateRight(x);
                    }
                    setColor(parentOf(x), BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    rotateLeft(parentOf(parentOf(x)));
                }
            }
        }
        root.color = BLACK;
}
```
判断要插入的元素是不是根节点，如果是根节点更新颜色为黑色，否则是红色，这里我们重点关注是红色的逻辑,先是一个while的死循环,结束条件是插入的元素
不为空并且不是root节点并且是插入元素父节点是红色。在while循环有两个大的分支判断。

- x的父节点是其祖父节点的左子节点:  
  &nbsp;&nbsp;&nbsp;&nbsp;获取变量y,y是祖父节点的右子节点，即是x的叔叔节点,这里又分两种情况:

  - 如果叔叔也是红色节点，根据第四章节分析的场景3.1父亲和叔叔为红色节点情况可知：  
  &nbsp;&nbsp;&nbsp;&nbsp;将叔叔节点和父节点设置为黑色节点，设置祖父节点为红色，祖父节点设置为当前节点，进行后续处理继续进行修复。  
  - 如果叔叔节点y是黑色的,进入下一个修复步骤：

  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果x是父节点的右子节点，就是第四章节分析的场景3.2.2LR型失衡,首先得到节点x的父节点并重新赋值给x,将父节点x
  &nbsp;&nbsp;&nbsp;&nbsp;进行左旋操作,调用的方法是rotateLeft,该方法的签名如下:
```java
 private void rotateLeft(Entry<K,V> p) {
      if (p != null) {
          Entry<K,V> r = p.right;
          p.right = r.left;
          if (r.left != null)
              r.left.parent = p;
          r.parent = p.parent;
          if (p.parent == null)
              root = r;
          else if (p.parent.left == p)
              p.parent.left = r;
          else
              p.parent.right = r;
          r.left = p;
          p.parent = r;
      }
  }
```
那我按照场景3.2.2图场景分析,获取p.right的代表变量r等于节点K,将K的左子树挂到p的右子树上，因为p要下沉成为K的左子树，设置K的左子树的父节点是F,
K的父指针指向F的原父节点,这样有助于K成为F的父节点，判断p.parent如果等于null,说明p是根节点，那么k成为新的根节点,否则更新F父节点的左子树为K
或者更新F父节点的右子树为K,最后将K的左子树变为F,F的父子树变为K,通过rotateLeft方法就把原先的父节点变为当前节点的子节点，但是还没有完事，因为此时相邻
节点还都是红色的,需要将节点P变红,K变黑，然后进行右旋平衡操作，调用的方法的是rotateRight,该方法的签名是:
```java
 private void rotateRight(Entry<K,V> p) {
        if (p != null) {
            Entry<K,V> l = p.left;
            p.left = l.right;
            if (l.right != null) l.right.parent = p;
            l.parent = p.parent;
            if (p.parent == null)
                root = l;
            else if (p.parent.right == p)
                p.parent.right = l;
            else p.parent.left = l;
            l.right = p;
            p.parent = l;
        }
}
```
此时传入的节点是P,把P的左子树赋值给变量l,把l的右子树赋值给P的左子树，判断如果l的右子树不为空，把l的右子树的parent设置为P,设置元素k的parent指向
P的parent,判断p.parent如果等于null,说明P是根节点，那么K成为新的根节点，否则更新P父节点的左子树为K或者右子树，最后将P变成K的右子树，P的父节点变成K。
- x的父节点是其祖父节点的右子节点:

&nbsp;&nbsp;这块的逻辑其实是跟上面的逻辑是对称操作，是上面的相反方向的操作，这里我就不分析了，感兴趣的同学可以自行分析一下。

### 5.2 查找(get方法)
&nbsp;&nbsp;该方法的签名如下:
```java
public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
}

final Entry<K,V> getEntry(Object key) {
        if (comparator != null)
           return getEntryUsingComparator(key);
        Objects.requireNonNull(key);
        Comparable<? super K> k = (Comparable<? super K>) key;
        Entry<K,V> p = root;
        while (p != null) {
          int cmp = k.compareTo(p.key);
          if (cmp < 0)
            p = p.left;
          else if (cmp > 0)
            p = p.right;
          else
           return p;
        }
        return null;
}
```
get方法逻辑比较简单,首先获取根元素赋值给变量p,然后根据传入的元素调用compareTo方法进行比较，根据结果和0比较，如果小于就把p的左子树赋值给p,如果大于就把
p的右子树赋值给p,如果等于直接返回元素p。