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
  
  以某个结点作为支点(pivot)，其父节点（子树的root）旋转为自己的左子树（左旋），pivot的原左子树变成原root节点的右子树，pivot的原右子树保持不变,我这里有张图方便理解

  ![img10.png](..%2Fimg%2Fimg10.png)

  从这张图可以看出,以18这个元素为pivot,把其父节点9旋转为自己的左子树,pivot的原左子树10变成root节点9的右子树

- **右旋:**

