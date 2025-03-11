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

| 对比维度 |     原生数值      |        ArrayList         |
|:-----|:-------------:|:------------------------:|
| 初始化	 | 必须显式指定长度（int[] arr = new int[10]） |   支持懒加载（默认构造器初始化为空数组）    |
| 扩容机制 |需手动创建新数组并拷贝数据	|   自动按1.5倍扩容（grow()方法）    
| 内存管理 |扩容后旧数组成为垃圾对象	| 内部elementData数组复用，减少GC压力 |

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

### 2.核心接口介绍
**2.1 List<E>接口的实现**

作为动态数组的核心契约，List 接口的实现是 ArrayList 的基础能力来源。

**(1) 动态扩容的 add 方法**
```java
// 添加单个元素（核心方法）
public boolean add(E e) {
    modCount++; // 结构性修改计数器
    add(e, elementData, size); // 内部方法
    return true;
}

private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length) // 容量已满
        elementData = grow();     // 扩容逻辑
    elementData[s] = e;          // 尾部插入
    size = s + 1;                // 更新逻辑大小
}
```
**设计特点：**

自动扩容：通过 grow() 方法实现 1.5 倍扩容（int newCapacity = oldCapacity + (oldCapacity >> 1)），平衡内存占用和扩容频率。

尾部插入优化：时间复杂度为 O(1)（不扩容时），相比 LinkedList 的中间插入更高效。

**(2) 随机访问的 get 方法**
```java
public E get(int index) {
    Objects.checkIndex(index, size); // JDK9+ 边界检查
    return elementData(index);       // 直接数组访问
}

// 无检查的底层访问（包级私有）
E elementData(int index) {
    return (E) elementData[index];
}
```
**性能优势：**

直接通过数组下标访问，时间复杂度为 O(1)，是 RandomAccess 接口的典型实现。

对比 LinkedList 的 get(int index)（需要遍历链表，时间复杂度 O(n)）。

**2.2 Clone接口实现**

ArrayList 实现了浅拷贝（Shallow Copy）的克隆机制。

**(1) clone() 方法源码**
```java
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size); // 复制数组
        v.modCount = 0; // 重置修改计数器
        return v;
    } catch (CloneNotSupportedException e) {
        throw new InternalError(e); // 理论上不会发生
    }
}
```
**关键点：**

浅拷贝：仅复制对象引用，不复制元素对象本身。

数组拷贝：Arrays.copyOf 复制原数组的有效部分（size 长度），避免保留多余容量。

## 三、 ArrayList的几个构造函数详解

**3.1 默认构造器：懒加载策略与10容量奥秘**

**源码实现**
```java
// JDK 21 源码
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
由此可以看出默认构造函数在初始化的时候被复制为空数组，不立即分配内存，这属于懒加载机制，
在首次调用 add() 时触发扩容，容量设为默认值 10：代码如下：
```java
private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow(); // 首次添加时扩容
    elementData[s] = e;
    size = s + 1;
}

private Object[] grow() {
    return grow(size + 1); // 最小容量为当前size+1
}

private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 首次扩容时使用默认容量10
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
}
```
**适用场景：**

临时存储：不确定初始数据量且可能长期空置的列表。

内存敏感环境：避免提前占用未使用的内存空间。

**潜在风险：**

多次扩容代价：若最终存储大量数据，频繁扩容可能导致性能损耗（如添加1000个元素需扩容 7次）。

**3.2 指定容量构造器：精准内存控制**

这个代码如下：
```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    }
}
```
可以看出根据调用者传递的容量初始化Object数组的elementData长度。这个构造函数有什么优点那？

**零扩容开销：** 若预设容量 ≥ 最终数据量，添加操作的时间复杂度稳定为 O(n)。

**空间最优化：** 无未使用槽位，内存利用率 100%。

**适用场景：**

大数据处理：已知数据规模（如分页查询每页固定100条）。

高频写入：实时日志收集等需要避免扩容延迟的场景。

**代码示例**
```java
// 预分配容量避免扩容
List<LogEntry> logs = new ArrayList<>(10000);
while (logSource.hasNext()) {
    logs.add(parseLog(logSource.next())); // 无扩容开销
}
```
**3.3 集合参数构造器：安全数据迁移**

这个有参构造函数的源码如下所示：
```java
public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a; // 直接复用数组（前提是原始集合为ArrayList）
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);
        }
    } else {
        elementData = EMPTY_ELEMENTDATA;
    }
}
```
**关键设计：**

高效复制：

若输入集合是 ArrayList，直接复用其底层数组（无拷贝开销）。

若非 ArrayList，通过 Arrays.copyOf 深度拷贝，时间复杂度 O(n)。

空集合处理：若输入集合为空，初始化 elementData 为空数组。

**安全机制：**

数据隔离：通过拷贝操作确保新列表与原集合无引用关联，避免意外修改。

类型安全：Arrays.copyOf 自动处理泛型数组类型转换问题。

**适用场景：**

集合转换：将其他集合类型（如 LinkedList、Set）转换为动态数组。

防御性拷贝：需要独立副本以避免共享数据修改的场景。

**3.4 三种构造函数的对比**

通过上面分析可以看出三种构造函数的各自设计特点以及适用场景，下面画个图表总结一下它们。

| 维度	 |     默认构造器     |  指定容量构造器	   | 集合参数构造器 |
|:---|:-------------:|:-----------:|:--------:
|内存分配时机 | 首次添加元素时（懒加载） |  初始化时立即分配	  | 初始化时根据输入集合大小分配
|初始容量	|0（首次扩容到10）	|    用户指定	    |等于输入集合的 size|
|适用场景	| 不确定数据量的临时存储	| 已知数据规模的预分配	 |集合转换|
|扩容开销	|可能多次扩容（最差O(log n)次）	|  无（若容量足够）	  | 无（容量精确匹配集合大小）|
|线程安全性	| 非线程安全	|   非线程安全	    | 非线程安全	|

## 四、核心方法源码解析
**1. ArrayList的add方法源码分析**

首先我们看该方法的签名,我用的是jdk21:
```java
public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}

private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)
        elementData = grow();
        elementData[s] = e;
        size = s + 1;
}
private Object[] grow() {
    return grow(size + 1);
}
private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        int newCapacity = ArraysSupport.newLength(oldCapacity,
        minCapacity - oldCapacity, /* minimum growth */
        oldCapacity >> 1           /* preferred growth */);
        return elementData = Arrays.copyOf(elementData, newCapacity);
    } 
    else {
    return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
}

public static int newLength(int oldLength, int minGrowth, int prefGrowth) {
       
    int prefLength = oldLength + Math.max(minGrowth, prefGrowth); 
    if (0 < prefLength && prefLength <= SOFT_MAX_ARRAY_LENGTH) {
    return prefLength;
    } else {
    // put code cold in a separate method
    return hugeLength(oldLength, minGrowth);
    }
}

private static int hugeLength(int oldLength, int minGrowth) {
    int minLength = oldLength + minGrowth;
    if (minLength < 0) {
        throw new OutOfMemoryError(
        "Required array length " + oldLength + " + " + minGrowth + " is too large");
    } 
    else if (minLength <= SOFT_MAX_ARRAY_LENGTH) {
      return SOFT_MAX_ARRAY_LENGTH;
    } else {
    return minLength;
    }
}
```
从add方法开始调用了重载方法，之后又调用了grow的重载方法，我一步步分析。
首先把modCount值自增加1,它记录了结构性修改的次数，主要是实现Fail-Fast机制，主要是迭代遍历时候防止被其他线程修改，如果修改抛出
ConcurrentModificationException，接着调用add方法,传入要添加的元素e,Object类型数组elementData,s代表集合中元素个数，首次添加时候是
0,第一次添加由于Object数组elementData的长度是0,前面也说了属于惰性的思想，会调用grow方法，grow又调用了重载还是把minCapacity的值设为1传入，把数组长度赋值
给oldCapacity,判断它是否大于0或者elementData不是空数组，很显然第一次添加不满足这个条件走到else语句，初始化数组长度是10，然后grow方法执行结束返回到add
方法，此时elementData数组下标0的元素是e,size的值1，执行结束。

由于数组初始化长度是10，当添加第十一个元素时候又走到grow(int minCapactiy)方法，此时判断oldCapacity是否大于0，
很显然是大于0，走到if语句中，此次调用了ArraysSupport.newLength方法，传入参数是oldCapacity=10，minCapacity - oldCapacity=11-10,
oldCapacity >> 1=10 >> 1,右移1位，得到的值是5,执行int prefLength = oldLength + Math.max(minGrowth, prefGrowth);
这里面的逻辑是新容量优先选择旧容量 + 首选增长量但如果首选增长量 < 最小增长量,则选择旧容量 + 最小增长量。如果prefLength小于Integer的最大长度-8返回
prefLength的值，若计算值超过SOFT_MAX_ARRAY_LENGTH,调用hugeLength方法，判断minLength<0,说明整数溢出抛出错误异常，
若必须扩容到 minLength 超过SOFT_MAX_ARRAY_LENGTH，则允许扩容到 Integer.MAX_VALUE。得到最终容量是之前的1.5倍，即新数组的容量是15，
调用Arrays.copyOf(elementData, newCapacity)方法复制数组，把elementData引用指向复制后的新数组，之前的旧的数组会被
GC垃圾收集器收集。这里有个问题需要思考一下为什么要扩容到1.5倍那，查阅相关资料得到这样的答案：

**ArrayList选择1.5倍扩容是为了在性能（减少扩容次数）和内存利用率(避免过多浪费)之间取得平衡。这使得ArrayList在大部分应用场景下能提供良好的性能和较低的内存开销。**

最后给大家出个ArrayList的add方法一个执行流程图方便理解：


![img3.jpg](..%2Fimg%2Fimg3.jpg)

**2. ArrayList的get方法源码分析**

get方法比较简单就是取得给定下标在数组中的指定位置,相信大家都能看懂，我还是给出get方法的签名：
```java
@Override
public E get(int index) {
    return a[index];
}
```
**3. ArrayList的contains方法源码分析**

contains方法的签名如下所示:
```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}

public int indexOf(Object o) {
    return indexOfRange(o, 0, size);
}

int indexOfRange(Object o, int start, int end) {
    Object[] es = elementData;
    if (o == null) {
        for (int i = start; i < end; i++) {
            if (es[i] == null) {
            return i;
            }
        }
    } else {
        for (int i = start; i < end; i++) {
            if (o.equals(es[i])) {
            return i;
            }
        }
    }
    return -1;
}
```
可以看出传入Object类型的对象，调用indexOf方法最终调用了indexOfRange方法，入参有Object类型 o，起始位置int类型start=0,结束位置是int类型容器中元素的个数end=size,
把elementData数组赋值给一个临时变量es,判断要包含的元素是否为null,如果为null,遍历从start-end下标值如果是null的返回下标值。

如果不是null,调用equals方法比较对象是否相同，如果相等返回下标，如果不相等返回-1，根据返回下标判断是否大于等于0，如果是返回true,代表包含，否则不包含,这里需要注意一点如果是自定义的引用类型应该重写equals方法，保证符合业务逻辑，只要内容相同就代表包含这个对象。

**4. ArrayList的remove方法源码分析**
remove方法有两个重载的方法,一个参数类型是int的,代表根据下标删除对象，一个参数类型是Object的，代表根据对象删除,我们一个个来看，首先看根据下标删除的方法签名
```java
public E remove(int index) {
    Objects.checkIndex(index, size);
    final Object[] es = elementData;

    @SuppressWarnings("unchecked") E oldValue = (E) es[index];
    fastRemove(es, index);

    return oldValue;
}

private void fastRemove(Object[] es, int i) {
    modCount++;
    final int newSize;
    if ((newSize = size - 1) > i)
    System.arraycopy(es, i + 1, es, i, newSize - i);
    es[size = newSize] = null;
}
```
根据传入的下标得到数组存储的值oldValue,然后调用fastRemove方法，modCount值自增1,定义变量newSize,
把size-1赋值给newSize,并且判断是否大于传入的下标，如果大于，说明是移除中间的某个下标元素，调用System.arraycopy方法把下标i后面元素移动到要删除下标位置上，这样做的目的是使数组地址连续， 最后把newSize的值赋值给size并且把size处元素置为null最终返回oldValue值。

下面我们来看看另一个重载方法remove的参数类型是Object的，方法签名如下:
```java
public boolean remove(Object o) {
    final Object[] es = elementData;
    final int size = this.size;
    int i = 0;
    found: {
        if (o == null) {
            for (; i < size; i++)
                if (es[i] == null)
                    break found;
        } else {
            for (; i < size; i++)
                if (o.equals(es[i]))
                    break found;
        }
        return false;
    }
    fastRemove(es, i);
    return true;
}
```
这个方法也比较简单，还是根据要删除对象，循环遍历比较equals方法如果有相等的就返回下标值，之后的调用和上面分析的逻辑一样，不在复述。

## 五、结束语

通过层层剖析ArrayList的源码，我们得以窥见Java集合框架的深邃智慧。Object[] elementData 的朴实无华，承载着动态扩容的灵活；size 字段的静默记录，维系着逻辑与物理的平衡；从默认构造器的懒加载策略到 grow()方法，每一行代码都在诉说着工程与艺术的交融。
ArrayList 的设计，是时间与空间的博弈，是安全与效率的权衡，更是开发者与机器的默契。它用最简洁的数组结构，支撑起高频查询的极致性能；以看似平凡的增删逻辑，诠释了算法优化的精妙。
然而，源码的魅力不止于理解，更在于启示。希望我们程序员都能写出高效的代码。