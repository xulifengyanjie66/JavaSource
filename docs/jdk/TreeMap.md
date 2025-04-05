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