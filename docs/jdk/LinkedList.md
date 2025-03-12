# LinkedList 底层源码深度解析

## 目录
- [1. 引言](#1-引言)
- [2. LinkedList 概述](#2-linkedlist-概述)
  - [2.1 类继承体系图](#21-类继承体系图)
  - [2.2 各个接口作用](#22-各个接口作用)
- [3. 与 ArrayList 的对比](#3-与-arraylist-的对比)
- [4. 底层数据结构](#4-底层数据结构)
 - [4.1 节点结构（Node 内部类）](#41-节点结构node-内部类)
- [5. 核心方法源码解析](#5-核心方法源码解析)
 - [5.1 add() 方法](#51-add-方法)
 - [5.2 remove() 方法](#52-remove-方法)
 - [5.3 get() 方法](#53-get-方法)
- [6. 迭代器实现](#6-迭代器实现)
 - [6.1 Iterator 迭代器](#61-iterator-迭代器)
 - [6.2 ListIterator 迭代器](#62-listiterator-迭代器)
- [7. 性能分析](#7-性能分析)
- [8. 线程安全性](#8-线程安全性)
- [9. 常见问题](#9-常见问题)
- [10. 总结](#10-总结)
- [11. 参考资料](#11-参考资料)

---

## 1. 引言
   在Java集合框架（Java Collections Framework, JCF）中，LinkedList 是一个常用的数据结构，它基于双向链表(Doubly Linked List)实现，适用于频繁的插入、删除操作。相较于ArrayList,LinkedList具有不同的性能特性和适用场景。因此，本文将深入分析 LinkedList 的实现原理、数据结构及其核心方法的源码，以帮助开发者更好地理解其底层逻辑。

**为什么选择分析 LinkedList？**

- **不同于 ArrayList，适合插入删除操作**

   ArrayList 依赖于动态数组存储元素，当插入或删除元素时，需要移动大量元素，而 LinkedList 采用双向链表结构，插入和删除操作仅需调整指针，时间复杂度为 O(1)（在已知节点的情况下）。

- **适用场景**
   
   频繁插入、删除元素（如队列、栈等结构）。
   不关心随机访问性能（LinkedList 随机访问的时间复杂度为 O(n)）。
   在 Deque（双端队列）或 Queue 场景下，如 LinkedList 实现了 Deque 接口，可用于双端队列操作。

- **JDK 版本说明**

   本次分析基于JDK21


---

## 2. LinkedList 概述

### 2.1 类继承体系图


类的继承结构图如下图所示：

![img4.png](..%2Fimg%2Fimg4.png)

### 2.2 各个接口作用

**1. List<E>（有序列表接口）**

- 作用：提供按索引存取元素的方法，实现线性表的基本功能。
- LinkedList 作为List的实现，可以存储有序的元素，并支持基于索引的操作。
- 关键方法：
```java
    boolean add(E e);         // 添加元素到链表末尾
    void add(int index, E e); // 在指定位置插入元素
    E get(int index);         // 获取指定索引的元素（时间复杂度 O(n)）
    E remove(int index);      // 删除指定索引的元素
    int indexOf(Object o);    // 获取元素第一次出现的位置
```
- 特点：

 get(int index) 由于 LinkedList 采用双向链表，需要遍历查找索引位置，时间复杂度为 O(n)（不像 ArrayList 是 O(1)）。
add/remove 在头部或尾部操作的时间复杂度是 O(1)。

**2. Deque<E>(双端队列接口)**

- 作用：允许两端进行插入和删除操作，支持队列和栈的特性。
- LinkedList作为Deque的实现，可以当作双端队列（Double-Ended Queue）或栈（Stack)来使用。
- 关键方法:
```java
    void addFirst(E e); // 在链表头部添加元素
    void addLast(E e);  // 在链表尾部添加元素
    E removeFirst();    // 移除头部元素
    E removeLast();     // 移除尾部元素
    E getFirst();       // 获取头部元素但不删除
    E getLast();        // 获取尾部元素但不删除
```
- 特点：

通过 addFirst/removeFirst 可以实现 FIFO 队列（先进先出，Queue）。
通过 addFirst/pop 可以实现 LIFO 栈（后进先出，Stack）。

**示例：LinkedList 作为队列**
```java
    Deque<Integer> queue = new LinkedList<>();
    queue.addLast(1);  // 入队
    queue.addLast(2);
    System.out.println(queue.removeFirst()); // 出队，输出 1

```
**示例：LinkedList 作为栈**
```java
    Deque<Integer> stack = new LinkedList<>();
    stack.addFirst(1); // 入栈
    stack.addFirst(2);
    System.out.println(stack.removeFirst()); // 出栈，输出 2
```
**3. Cloneable（可克隆接口）**
- 作用：表示 LinkedList 可以被克隆，支持 clone() 方法。
- 关键方法
```java
 protected Object clone() throws CloneNotSupportedException;
```
- 特点：
- clone() 方法执行的是浅拷贝，不会复制存储的对象本身，而是复制引用。
- 示例：
```java
LinkedList<String> list1 = new LinkedList<>();
list1.add("A");
list1.add("B");

LinkedList<String> list2 = (LinkedList<String>) list1.clone();
list2.add("C");

System.out.println(list1); // [A, B]
System.out.println(list2); // [A, B, C]
```
**4. Serializable（可序列化接口)**
- 作用：使 LinkedList 支持序列化，可以将 LinkedList 对象转换为字节流，进行持久化存储或网络传输。

- 实现原理： LinkedList 内部有 private static final long serialVersionUID = 876323262645176354L; 用于版本控制。
- 关键方法：
```java
private void writeObject(ObjectOutputStream s) throws IOException;
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException;
```
- writeObject() 和 readObject() 通过遍历链表来保存和恢复节点数据。

**示例：序列化与反序列化**
```java
import java.io.*;

LinkedList<String> list = new LinkedList<>();
list.add("Hello");
list.add("World");

// 序列化
try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("list.ser"))) {
    out.writeObject(list);
}

// 反序列化
try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("list.ser"))) {
    LinkedList<String> newList = (LinkedList<String>) in.readObject();
    System.out.println(newList); // [Hello, World]
}

```
**5. AbstractSequentialList<E>（抽象顺序列表）**

- 作用：LinkedList 继承了AbstractSequentialList，它是 List 的一个抽象实现，提供了基于顺序访问（Sequential Access）的列表操作。
- 关键方法：
```java
public E get(int index);
public ListIterator<E> listIterator(int index);
```
- 特点：
AbstractSequentialList 通过 listIterator() 提供了顺序遍历能力。
LinkedList 必须实现 listIterator(int index) 方法来提供双向遍历能力。

---

## 3. 与 ArrayList 的对比
- 内存结构差异（数组 vs 链表节点）
- 时间复杂度对比（插入/删除/查找）
- 适用场景分析

---

## 4. 底层数据结构

### 4.1 节点结构（Node 内部类）
```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    // 构造方法...
}
