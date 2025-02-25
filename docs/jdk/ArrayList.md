# 深入剖析Java ArrayList源码
## 一、引言
### 1.为什么选择ArrayList？
这里我想从两个方面分析，一是它是在什么场景下使用，二是它比数组的优势在哪里，下面我们一个个分析

**1.1开发中的高频使用场景**

ArrayList作为Java集合框架中最常用的动态数组实现，其高频使用场景与其设计特性密切相关。以下是实际开发中最典型的应用场景：

(1) 数据动态增长需求
场景示例：
用户注册时批量导入数据、日志系统收集动态生成的日志条目、电商购物车商品数量变化等。
```java
// 数据动态增长示例
List<String> logs = new ArrayList<>();
while (系统运行中) {
    String log = 生成日志();
    logs.add(log); // 无需关心底层数组扩容
}
```
核心优势：
ArrayList 的自动扩容机制（grow()方法）让开发者无需预先计算数据规模，规避了传统数组的定长限制。即使预估错误，也不会导致 ArrayIndexOutOfBoundsException。

(2) 临时数据容器 场景示例:解析 CSV/JSON 数据、网络请求结果反序列化、数据库查询结果集映射。
```java
// JSON解析示例
List<Map<String, Object>> tempList = new ArrayList<>();
while (jsonParser.hasNext()) {
    tempList.add(parseItem(jsonParser));
}
// 使用后快速释放
tempList.clear(); // 清空但不回收数组，复用内存空间
```
内存管理优势：
ArrayList的clear()方法仅将size置零，保留底层数组（elementData）避免频繁 GC。适合短期存活对象，减少内存分配开销。

**1.2对比数组的灵活性优势**

通过一个表格详细对比ArrayList与原生数组的差异

| 对比维度 |     原生数值      |              ArrayList |
|:-----|:-------------:|-----------------------:|
| 初始化	 | 必须显式指定长度（int[] arr = new int[10]） |    支持懒加载（默认构造器初始化为空数组） |
| 扩容机制 |需手动创建新数组并拷贝数据	|    自动按1.5倍扩容（grow()方法） 
| 内存管理 |扩容后旧数组成为垃圾对象	|内部elementData数组复用，减少GC压力 |

代码对比：
```java
// 原生数组扩容
int[] arr = new int[10];
int[] newArr = new int[arr.length * 2];
System.arraycopy(arr, 0, newArr, 0, arr.length);
arr = newArr;

// ArrayList扩容（完全封装，用户无感知）
List<Integer> list = new ArrayList<>();
list.add(1); // 首次添加触发扩容至10
```
## 二、ArrayList核心设计
### 1.类结构解剖
我们先看ArrayList的类结构继承图

![img2.png](..%2Fimg%2Fimg2.png)