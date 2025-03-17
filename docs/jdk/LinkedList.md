# LinkedList 底层源码深度解析

## 目录
- [1. 引言](#1-引言)
- [2. LinkedList 概述](#2-linkedlist-概述)
  - [2.1 类继承体系图](#21-类继承体系图)
  - [2.2 各个接口作用](#22-各个接口作用)
- [3. 与 ArrayList 的对比](#3-与-arraylist-的对比)
- [4. 底层数据结构](#4-底层数据结构)
- [5. 核心方法源码解析](#5-核心方法源码解析)
  - [5.1 add() 方法](#51-add-方法)
  - [5.2 add(int,Object) 方法](#52-addint-index-e-element方法)
  - [5.3 get() 方法](#53-getint-index方法)
- [6. 迭代器实现](#6-迭代器实现)
 - [6.1 Iterator 迭代器](#61-listiterator迭代器)
- [7. 总结](#7-总结)

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
**1.List&lt;E&gt;（有序列表接口）**

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

**2.Deque&lt;E&gt;（双端队列接口）**

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
**3.Cloneable（可克隆接口）**
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
**4. Serializable**（可序列化接口)

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
**5. AbstractSequentialList&lt;E&gt;**（抽象顺序列表）

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
  **1. 底层数据结构**

   - ArrayList：基于动态数组实现，底层是一个 Object[] 数组，元素按索引存储，支持随机访问。
   - LinkedList：基于双向链表实现，每个元素（节点）包含数据和前后指针，存储在非连续的内存空间。

  **2. 时间复杂度对比**

  | 操作类型 |   ArrayList    |        LinkedList        |
  |:-----|:--------------:|:------------------------:|
  | 随机访问 (get(int index))	 |  O(1)(直接索引访问)  |      O(n)(从头/尾遍历链表)      |
  | 插入 (add(E e)) | O(1) / O(n)（尾部 O(1)，中间 O(n)）	 |   O(1) / O(n)（头尾 O(1)，中间 O(n)）    
  | 删除 (remove(int index)) | O(n)（元素后移）	  | O(1) / O(n)（头尾 O(1)，中间 O(n)） |
  | 遍历 | O(n)（数组连续，CPU 缓存友好）	  | O(n)（指针访问，不支持缓存优化） |

**3. 空间开销**

   - ArrayList
   由于底层是数组，它会预留额外的容量，通常是当前容量的 1.5 倍（newCapacity = oldCapacity + (oldCapacity >> 1)）。
   数据连续存储，没有额外的指针开销。
   - LinkedList
   每个节点存储 两个额外的指针（prev 和 next），导致空间额外开销较大。
   碎片化严重，数据分散在不同的内存位置，不利于 CPU 缓存优化。

- 结论：

    ArrayList 占用更少的额外内存，更适合大批量数据存储。
    LinkedList 需要额外的指针存储，当数据量大时，占用的内存更多。

**4. 遍历性能**

   - ArrayList
   由于底层是数组，支持 CPU 缓存优化（Cache Friendly）。
   遍历速度快（O(n)）。
   - LinkedList
   由于节点分散在内存中，指针访问会有额外开销（Cache Unfriendly）。
   遍历时需要频繁的指针跳转，效率较低。

- 结论：

    遍历速度：ArrayList 远快于 LinkedList。
---

## 4. 底层数据结构
```java
public class LinkedList<E>
        extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    transient int size = 0;

    /**
     * Pointer to first node.
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     */
    transient Node<E> last;
}

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```
LinkedList类的成员属性有指向内部类的Node节点，分别指向链表的头部和尾部,内部类Node
可以看出除了存储元素类型为E的元素以外，它们之间也有指向下一个节点和上一个节点的引用,总的来说，first和last是用来存储链表的头节点和尾节点的引用。
next和prev是用于存储链表中各个节点的前后关系，它们保证了链表的双向遍历能力。

---
## 5. 核心方法源码解析

### 5.1 add() 方法

该方法签名如下：
```java
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    
    void linkLast(E e) {
      final Node<E> l = last;
      final Node<E> newNode = new Node<>(l, e, null);
      last = newNode;
      if (l == null)
        first = newNode;
      else
        l.next = newNode;
      size++;
      modCount++;
   }
```
这里我以添加两次元素为例说明内存节点的变化情况：

**第一次:** 比如添加了一个元素3,调用linkLast方法,把指向链表末尾的last属性赋值给一个临时变量l,第一次添加指定是
l=last =null,新创建一个Node对象，把l赋值给Node成员属性prev,把e赋值给成员属性item,把null赋值给成员属性next,
把last指向新创建的元素，判断临时变量l是否等于null,如果是null,说明是首次添加元素，此时first也指向新创建的节点，把size值加1,modCount值加1，此时内存结构是如果所示：

![img5.jpg](..%2Fimg%2Fimg5.jpg)

**第二次:** 比如添加了一个元素4,调用linkLast方法，同样把last赋值给临时变量l,但是此时l不是为null,而是指向元素3，这一点相信大家可以理解,
创建一个新元素，新元素的prev指向元素3，next指向null,把last引用指向元素4，代表是最后一个元素，l.next指向新元素，即元素3的next指向新元素，
把size值加1,modCount值加1，此时内存结构是如果所示:

![img6.jpg](..%2Fimg%2Fimg6.jpg)

通过这两次添加元素3，元素4都添加到链表上，总体结构是3是第一个元素，4是最后一个元素，他们之间通过prev,next互相指向，而且first指向元素3，last指向元素4，
到这里add方法流程就分析完成了。

### 5.2 add(int index, E element)方法

下面我们来分析一下add的一个重载方法，根据下标插入元素的源码分析，该方法的签名如下所示：

```java
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}

Node<E> node(int index) {

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
        x = x.next;
        return x;
    } 
    else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
        x = x.prev;
        return x;
    }
}

void linkBefore(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
      pred.next = newNode;
    size++;
    modCount++;
}

```

该方法的主要作用是在指定索引位置插入元素，比如在索引位置1插入元素，那么原先索引位置是1的元素往后移动，新插入的元素的next指向原来位置元素，原来位置元素的prev
指向新元素，下面我将详细分析其源码。

调用checkPositionIndex校验传入的index下标是否小于0或者大于size值，如果是抛出下标越界异常IndexOutOfBoundsException，判断下标是否等于size值，如果是调用linkLast方法把元素添加到尾部，这个方法在add源码中已经分析了，此处不在赘述，
如果不是调用linkBefore方法，该方法中又调用了node方法，从上面给出的源码可以看出有两个分支条件。

- 如果index在前半部分（index < size/2），则从头结点开始查找。
- 否则，从尾结点开始倒着查找，提高性能。

**此处使用了二分优化查找的思想，提高遍历性能。**

分析从头查找的逻辑，先把首节点first赋值给一个临时变量x,使用for循环以当前传入的下标为终止条件，如果小于就一直取得next节点赋值给x，直到条件不成立最终返回x。

分析从尾查找的逻辑, 先把尾节点last赋值给一个临时变了x,使用for循环依次从size-1处递减直到小于传入的下标为终止条件，如果条件成立一直取得prev节点赋值给x,
直到条件不成立最终返回x。

上面找到要插入的元素，调用linkBefore在其之前执行插入操作,先找到succ位置的前驱节点pred,创建新节点newNode,使其prev指向pred,next指向succ。
更新pred.next和succ.prev，完成插入,如果 pred 为空,说明succ原来是first,新节点需要变成first,至此根据下标插入元素的源码就分析完成了。

### 5.3 get(int index)方法

该方法的签名如下:

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

Node<E> node(int index) {
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
        x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
        x = x.prev;
        return x;
    }
}
```
该方法同样是采用有二分优化查找的思想，从头或者从尾开始查找，直到找到符合条件的Node元素，然后获取里面的item属性值，这个方法比较简单,不在赘述。

### 5.4 remove(int index)方法

该方法的签名如下：

```java
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

Node<E> node(int index) {
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
        x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
        x = x.prev;
        return x;
    }
}

E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }
    x.item = null;
    size--;
    modCount++;
    return element;
}
```
可以看出根据索引删除元素方法先是调用了node方法获取元素，这个方法在前面已经分析过了，就不再分析，我们主要看unlink方法。

首先获取item属性值，即里面存储的元素值赋值给element变量，取得后置节点和前置节点分别赋值给next和prev变量，判断prev是否等于null,如果是null,说明
删除的元素是头节点，那么头节点删除以后first应该指向next节点，如果不是头节点，把prev的next指向要删除元素的next节点，断开删除元素指向前置节点的引用。

还有一直情况是判断next是否等于null,如果是null,说明删除的元素是尾节点，那删除尾节点以后last应该指向它的前置节点prev,如果不是尾节点，把next的prev指向
要删除元素的prev,同时断开删除元素指向后置节点的引用，把删除元素的属性值置为null,size个数减1操作，modCount加1，最后返回临时变量element。

大家可能觉得这个流程有点蒙，下面我给出一个流程图，如下图所示：

![img7.jpg](..%2Fimg%2Fimg7.jpg)

### 5.5 remove(Object o)方法

该方法的签名如下：

```java
public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
}
```
判断要删除的对象是否等于null,如果是null,从链表的头部开始遍历，如果有是null的节点就调用unlink方法移除该元素，这个方法跟上面调用的是同一个方法，如果不是null还是要遍历所有链表，
调用对象的equals方法判断是否相等，如果相等同样调用unlink方法移除该元素。

## 6. 迭代器实现

### 6.1 Listiterator迭代器

**1. ListIterator 概述**

这里我介绍一下Listiterator迭代器概念以及它能提供的功能。

ListIterator是Iterator接口的扩展，专门用于List集合类型，它能够在集合中进行双向遍历。与 Iterator 只能向前遍历不同，ListIterator 允许我们从当前元素开始向前和向后遍历。除此之外，它还提供了以下几个主要的功能：

- **前向遍历**：使用 next() 方法逐个返回元素。
- **后向遍历**：使用 previous() 方法逐个返回元素（反向遍历）。
- **修改元素**：使用 set(E e) 方法修改当前元素。
- **添加元素**：使用 add(E e) 方法在当前位置插入一个新元素。
- **删除元素**：使用 remove() 方法删除当前元素。

**2. ListIterator使用示例**

- 向前遍历

```java
import java.util.*;

public class ListIteratorForwardExample {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>(Arrays.asList("A", "B", "C", "D"));

        ListIterator<String> listIterator = list.listIterator();

        while (listIterator.hasNext()) {
            System.out.println("向前遍历：" + listIterator.next());
        }
    }
}
```
输出结果是：
```css
向前遍历：A
向前遍历：B
向前遍历：C
向前遍历：D
```
- 向后遍历
```java
import java.util.*;

public class ListIteratorBackwardExample {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>(Arrays.asList("A", "B", "C", "D"));

        ListIterator<String> listIterator = list.listIterator(list.size());

        while (listIterator.hasPrevious()) {
            System.out.println("向后遍历：" + listIterator.previous());
        }
    }
}
```
输出结果是：

```css
向后遍历：D
向后遍历：C
向后遍历：B
向后遍历：A
```
- 修改元素

```java
import java.util.*;

public class ListIteratorModifyExample {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>(Arrays.asList("A", "B", "C", "D"));

        ListIterator<String> listIterator = list.listIterator();

        while (listIterator.hasNext()) {
            String current = listIterator.next();
            if (current.equals("B")) {
                listIterator.set("Z");  // 修改元素 B 为 Z
            }
        }

        System.out.println(list);  // 输出：[A, Z, C, D]
    }
}
```
- 插入元素

```java
import java.util.*;

public class ListIteratorAddExample {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>(Arrays.asList("A", "B", "C", "D"));

        ListIterator<String> listIterator = list.listIterator();

        while (listIterator.hasNext()) {
            String current = listIterator.next();
            if (current.equals("B")) {
                listIterator.add("Z");  // 在 B 后插入 Z
            }
        }

        System.out.println(list);  // 输出：[A, B, Z, C, D]
    }
}

```

## 7. 总结

本文深入分析了LinkedList与ArrayList在数据结构上的不同，探讨了它们的适用场景及时间复杂度对比。LinkedList作为一个基于双向链表的数据结构，在插入和删除操作方面表现优异，特别适用于频繁插入、删除的场景，而 ArrayList 则在随机访问方面更具优势。

在源码分析部分，我们详细解读了LinkedList的底层实现，深入解析了add()方法的多个重载版本，了解了元素如何被插入到链表的不同位置。同时，我们分析了 get() 方法的源码，理解了 LinkedList 如何通过遍历获取元素，并探讨了 remove() 方法的具体实现，了解了删除操作如何调整链表的指针来维护数据结构的完整性。

此外，本文还对 ListIterator 进行了详细介绍。ListIterator 作为 Iterator 的增强版，支持双向遍历、元素修改、插入和删除，在操作 LinkedList 时尤其高效，能够充分发挥链表结构的特点。

通过这篇文章，相信你对 LinkedList 的内部实现有了更深入的理解，并能更合理地选择 LinkedList 或 ArrayList，在合适的场景下提高程序的性能和可读性。希望这篇文章能帮助你在 Java 集合框架的学习和应用中更进一步！