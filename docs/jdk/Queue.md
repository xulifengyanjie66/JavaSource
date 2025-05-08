# Java Queue简介

- [1. 简介](#1简介)
- [2. Queue核心概念](#2queue核心概念)
    - [2.1 FIFO原则](#21-fifo原则)
- [3. Queue实现类](#3queue实现类)
    - [3.1 LinkedList](#31-linkedlist)
    - [3.2 ArrayBlockingQueue](#32-arrayblockingqueue)
    - [3.3 PriorityQueue](#33-priorityqueue)
- [4. 核心操作方法对比](#4核心操作方法对比)
    - [4.1 插入操作对比](#4核心操作方法对比)
    - [4.2 移除操作对比](#42-移除操作对比)
- [5.使用场景](#5使用场景)
- [6.Queue vs ArrayList](#6queue-vs-arraylist)
    - [6.1方法差异详解](#61方法差异详解)
    - [6.2内存使用对比](#62内存使用对比)
- [7. 总结对比](#7总结对比)

## 1.简介
Queue（队列）是Java集合框架中表示先进先出(FIFO)数据结构的接口。它扩展自`Collection`接口，主要应用于需要有序处理的场景：
- 生产者-消费者模式
- 任务调度系统
- 异步消息传递
- 算法实现（如BFS）

与普通集合的区别：
```java
    // 队列的典型声明
    Queue<String> queue = new LinkedList<>();
    
    // 普通List声明
    List<String> list = new ArrayList<>();
```
## 2.Queue核心概念

### 2.1 FIFO原则
- 元素按插入顺序处理
- 先进入的元素最先被访问（poll/peek）
- 例外情况：PriorityQueue使用自然排序

## 3.Queue实现类

### 3.1 LinkedList

- 基于双向链表实现
- 支持null元素
- 线程不安全

### 3.2 ArrayBlockingQueue

- 固定大小的阻塞队列
- 内部使用ReentrantLock
- 支持公平锁策略
- 典型用法：
```java
BlockingQueue<Integer> bq = new ArrayBlockingQueue<>(10);
bq.put(1);  // 阻塞插入
int num = bq.take();  // 阻塞获取
```
### 3.3 PriorityQueue

- 基于堆结构（默认最小堆）
- 元素必须实现Comparable接口

- 时间复杂度：
  - 插入：O(log n)
  - 移除：O(log n)
  - 查看：O(1)
  
## 4.核心操作方法对比
### 4.1 插入操作对比
| 方法      |             队列满时候行为              |           返回值            |          抛出异常            |
|:--------|:--------------------------------:|:------------------------:|:------------------------:|
| add()	  |      IllegalStateException       |   boolean    |✓ |
| offer() |            立即返回false	            |   boolean    |	✗|

### 4.2 移除操作对比
| 方法      |             队列空时行为              | 返回值 |          抛出异常            |
|:--------|:-------------------------------:|:---:|:------------------------:|
| remove()	  |     NoSuchElementException      |  E  |✓ |
| poll() |            立即返回null	            |  E  |	✗|

# 5.使用场景
**适合Queue的场景**

1.订单处理系统
```java
Queue<Order> orderQueue = new ConcurrentLinkedQueue<>();
// 生产者
orderQueue.offer(new Order(...));
// 消费者
while(!orderQueue.isEmpty()) {
    processOrder(orderQueue.poll());
}
```
2.线程池任务队列
```java
new ThreadPoolExecutor(
    5, 10, 60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100)  // 使用队列管理任务
);
```
# 6.Queue vs ArrayList
## 6.1方法差异详解
**Queue特有方法：**
```java
// 安全插入
if(queue.offer(element)) {
    // 处理成功插入逻辑
}

// 批量操作
drainTo(Collection<? super E> c);  // 转移元素
```
**ArrayList核心方法：**
```java
// 中间插入
list.add(0, element);  // 导致所有元素后移

// 范围操作
List<String> sub = list.subList(1, 4); 
```
## 6.2内存使用对比
- **ArrayList：**
  - 连续内存空间
  - 默认初始容量10
  - 扩容机制：每次增长50%
- **LinkedList：**
  - 每个元素需要额外存储前后指针
  - 每个节点内存开销：12字节（header）+ 4*2（指针）+ 数据
  - 更适合存储大对象

# 7.总结对比
| 维度      |                Queue                | ArrayList |
|:--------|:-----------------------------------:|:---------:|
| 核心功能	   |               顺序处理机制	               |  动态数组存储   |
| 访问模式	   |              受限端点访问		               |   全索引访问   |
| 线程安全    |             有专门并发实现			              |   需外部同步   |
| 性能特点	   |             高效入队/出队			              |  快速随机访问   |
| 内存效率		  |            节点存储有额外开销				            | 连续存储空间高效  |
| 使用场景			 |            任务队列/消息传递				            |数据集存储/快速查找 |

选择建议：

- 95%的顺序处理场景使用Queue

- 需要索引操作时使用ArrayList

- 多线程环境优先选择并发Queue实现