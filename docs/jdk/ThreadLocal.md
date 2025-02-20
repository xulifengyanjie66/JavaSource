# ThreadLocal源码分析

## 1. 引言

### 1.1 什么是ThreadLocal?
ThreadLocal是Thread的局部变量，用于不同线程之间的数据隔离。每个线程都可以通过ThreadLocal获取到属于自己的数据副本，这样各个线程之间的数据不会互相干扰。

### 1.2 ThreadLocal的应用场景
ThreadLocal主要用于多线程环境下保持数据的独立性，不同线程的数据互不影响。通过ThreadLocal，每个线程可以拥有独立的变量副本，避免了在多线程环境中共享数据带来的数据竞争和线程安全问题。

## 2. ThreadLocal的源码分析

在分析ThreadLocal源码之前，我们需要了解ThreadLocal和Thread类、ThreadLocalMap、Entry、WeakReference之间的关系。

### 2.1 ThreadLocalMap与ThreadLocal类的关系
ThreadLocalMap是ThreadLocal的一个静态内部类,我给出一段部分ThreadLocal的代码：
```java
public class ThreadLocal<T> {

      static class ThreadLocalMap {
          
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

      
        private static final int INITIAL_CAPACITY = 16;

      
        private Entry[] table;

        
        private int size = 0;

      
        private int threshold; // Default to 0

      
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
        
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

      
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }

       
        private ThreadLocalMap() {
        }

       
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
        
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (Entry e : parentTable) {
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
        
        int size() {
            return size;
        }
        
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.refersTo(key))
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
        
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                if (e.refersTo(key))
                    return e;
                if (e.refersTo(null))
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

        private void set(ThreadLocal<?> key, Object value) {
            
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.refersTo(key)) {
                    e.value = value;
                    return;
                }

                if (e.refersTo(null)) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
        
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.refersTo(key)) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
        
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;
            
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.refersTo(null))
                    slotToExpunge = i;
            
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                if (e.refersTo(key)) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }
                
                if (e.refersTo(null) && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
        
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
        
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.refersTo(null)) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }

        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }
        
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (Entry e : oldTab) {
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
        
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.refersTo(null))
                    expungeStaleEntry(j);
            }
        }
    }
}
```

### 2.2 ThreadLocalMap与Entry之间关系
由上面的代码可以看出ThreadLocalMap内部又有一个静态内部类Entry,他继承了弱引用WeakReference，并指定泛型ThreadLocal,这表明如果ThreadLocal没有强引用指向将会被垃圾回收器回收。而对于Thread类，有个成员属性ThreadLocalMap，他是每个线程独有的，线程通过这个ThreadLocalMap达到不同线程进行数据隔离的。

### 2.3 ThreadLocalMap中成员属性参数的含义
知道了上面各个类之间的关系,我们还要了解一下ThreadLocalMap的成员属性的含义，以便我们可以更好的看懂源码。

1.private Entry[] table;声明一个类型是Entry的数组。

2.private static final int INITIAL_CAPACITY = 16; 代表table指向的Entry[]数组初始化数组大小为16。

3.private int size = 0;代表table数组中已经存在的元素数量。

4.private int threshold;代表扩容的参数，首次设置为10，threshold = 16* 2 / 3。

5.private final int threadLocalHashCode = nextHashCode(),它的值是nextHashCode.getAndAdd(HASH_INCREMENT),nextHashCode是一个AtomicInteger类型，HASH_INCREMENT值固定是0x61c88647，该成员变量的作用是用于为每个ThreadLocal实例生成唯一的哈希码。

### 2.4 ThreadLocal的set方法源码分析
set方法的源码如下所示:
```java
private void set(Thread t, T value) {
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```
从上面的代码可以看出这块分为两个分支，一个是当前线程的Thread的ThreadLocalMap不为空，一个为空，那首次进入的时候指定是为空的，那么我们先分析createMap(t,value)的源码。
#### 2.4.1 createMap方法源码分析
createMap方法的源码如下所示：
```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
这个方法给Thread的成员变量threadLocals即ThreadLocal.ThreadLocalMap threadLocals赋值，调用ThreadLocalMap的构造方法，代码如下：
``` java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
table = new Entry[INITIAL_CAPACITY];
int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
table[i] = new Entry(firstKey, firstValue);
size = 1;
setThreshold(INITIAL_CAPACITY);
}
```
我们来看看ThreadLocalMap的构造函数都做了什么，首先先创建了一个16个长度的Entry的数组，通过threadLocalHashCode按位与操作获取获取一个i，这个i决定了在数组中的什么位置上存储，在数组的i位置创建一个Entry对象，其中firstKey参数就是ThreadLocal实例对象，firstValue代表要存储的值，把size值赋值为1，设置hreshold = 16* 2 / 3，即是10。
createMap方法还是比较简单的就是一些初始化操作，相信大家都能看得懂。

#### 2.4.2 map.set(this, value)方法源码分析

如果第二次再调用ThredLocal的set方法,先获取ThreadLocalMap对象,由于此时该对象不为空，就会调用该对象的set方法，该方法的代码如下：
``` java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.refersTo(key)) {
            e.value = value;
            return;
        }

        if (e.refersTo(null)) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
同样根据ThreadLocal对象获取hashCode值在与上数组长度减去1得到下标,得到table数组中的存储位置，接着执行一个for循环，把tab[i]做为初始值，判断tab[i]位置是否有元素存在，这里先分析存在的情况，什么时候会存在那，就是同一个ThreadLocal对象调用了多次set方法，计算出的i值一直都是相同的，那么tab[i]值就不为空，接着调用Entry对象的refersTo方法，传入当前ThreadLocal对象，判断弱引用持有的对象是否是当前ThreadLocal对象，如果是把Entry的value成员属性替换设置为新传入的value值，然后方法结束执行;
还有一种情况就是就是发生了hash冲突，ThreadLocal对象不同，但是得到的下标相同，所以for循环的if条件都不成立，此时会新建一个Entry对象并把size值加一,这里加入发生冲突的i=3,那么for循环的条件都执行完成以后，i=4,为什么那，因为这段代码 e = tab[i = nextIndex(i, len)]，执行了i=nextIndex,该方法签名如下：
``` java
 private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
}
```
可以看出判断当前i+1如果小于数组长度就返回i+1,否则返回0，那么我们假如没超过数组长度那么加一就是返回4，不同的冲突元素会放在数值不同位置这与HashMap发生冲突时候不同，不是放在链表而是放在不同数组位置，这叫做开放地址法解决冲突，感兴趣的小伙伴可以自己查询一下这个解决冲突的原理。

回过头我们再看for循环的if语句中的一段逻辑，代码如下
```java
 if (e.refersTo(null)) {
            replaceStaleEntry(key, value, i);
            return;
}
```
如果当前元素强引用为空，即没人指向它，由于ThreadLocal是弱引用会被回收，refersTo判断等于null,此时调用replaceStaleEntry方法进行处理，该方法的签名如下:
```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.refersTo(null))
            slotToExpunge = i;
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        if (e.refersTo(key)) {
            e.value = value;
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        if (e.refersTo(null) && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```
replaceStaleEntry() 方法并非简单地使用新entry替换过期entry，而是从过期entry所在的slot（staleSlot）向前、向后查找过期entry，并通过slotToExpunge 标记过期entry最早的index，最后使用cleanSomeSlots() 方法从slotToExpunge开始清理过期entry。先是一个for循环初始化条件是i=prevIndex(staleSlot, len),方法prevIndex的方法签名是这样的：
```java
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```
如果i-1值大于0返回i-1的值，小于返回len-1值，计算tab[i]值，如果tab[i]不为空判断当前元素的ThreadLocal是否已经被GC回收，如果被回收了，把slotToExpunge设置为当前循环的下标，继续向前操作直到table中的元素为null,slotToExpunge值变成最后一个tab[i]不为空的下标值。

接着执行在for循环执行nextIndex方法，该方法在上面已经说过了，如果小于数组长度，返回当前下标加一的值，如果大于数组长度就从0开始遍历，每次遍历碰到元素不为空,由于开放定址法，可能相同的key存放于预期的位置（staleSlot）之后,即如果算出当前staleSlot=7，那么冲突的元素有可能在位置8上，所以此处判断元素是否和传递ThreadLocal是否相等，如果遇到相同的key，则更新value，并交换索引staleSlot与索引i的entry，判断slotToExpunge == staleSlot是否相等，如果相等说明向前遍历的没找到过期元素，更新slotToExpunge为当前索引下标值。从slotToExpunge开始，清理一些过期entry，这是会调用到方法expungeStaleEntry,该方法签名如下:
```java
 private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
}
```
这个方法举个上面说的例子，比如下标7上ThreadLocal被GC清除了，但是在下标8的位置上找到了要放的ThreadLocal,那么就把这个ThreadLocal放在下标为7的位置，把8的下标位置元素Entry的key和value都置为null,便于垃圾回收。
这块可以画个图理解一下，
在执行代码tab[i] = tab[staleSlot]; tab[staleSlot] = e;交换前，内容结构图假如是这样的:
![img1.jpg](..%2Fimg%2Fimg1.jpg)
此时交换完成以后,下标7即staleSlot=7位置是元素e,下标8的ThreadLocal为null,此时执行方法expungeStaleEntry就清除8下标Entry对应的key和value,便于GC进行垃圾回收。
然后把size做减一操作，接着执行for循环把下标继续加1，判断如果ThreadLocal为空，同样把Entry对应的key,value设置为null,size减一，如果不为空取出ThreadLocal的下标，
判断如何当前元素的下标和计算出来的下标不一样，把当前下标的对应元素设置为null,循环while判断tab[h]如果为空，把元素放到对应位置上，最终返回i的值，也就是前面for循环e = tab[i]) = null条件。

expungeStaleEntry方法执行完成以后返回到replaceStaleEntry接着又调用了cleanSomeSlots方法,该方法签名如下:
```java
private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.refersTo(null)) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
}
```
这段代码其实也是调用expungeStaleEntry方法进行清除操作，只不过while循环条件限制为n >>>= 1,实现约 log2(n) 次探测,每次都是折半的,
目的是避免全表扫描，举个例子，
假设哈希表长度为 16，初始 n = 4，探测步骤如下：
初始状态：i = 3, n = 4
循环 4 次（4 → 2 → 1 → 0）
第一次循环：
i = 4，发现无效 Entry
设置 n = 16，触发 expungeStaleEntry(4)
后续循环：
n 变为 8 → 4 → 2 → 1 → 0（共 5 次额外探测）
总探测次数 = 初始 4 次 + 重置后 5 次 = 9 次（远小于全表 16 次）

至此ThreadLocal的set源码算是分析清楚了，这里面expungeStaleEntry方法和cleanSomeSlots方法比较难理解，需要大家有点空间想象能力。

### 2.5 ThreadLocal的get方法源码分析
我们先看看get方法的签名:
```java
private T get(Thread t) {
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T) e.value;
            return result;
        }
    }
    return setInitialValue(t);
}
```
从当前线程中取出ThreadLocalMap,如果ThreadLocalMap不为空，调用getEntry方法，参数是ThreadLocal对象，getEntry方法签名如下:
```java
 private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.refersTo(key))
                return e;
            else
                return getEntryAfterMiss(key, i, e);
}
```
计算出ThreadLocal对象的下标，通过下标获取数组中的Entry对象，如果Entry对象不为空并且传入的ThreadLocal和查询出的Entry的key一样直接返回Entry对象,
如果不一样说明发生了冲突当前的ThreadLocal被存在别的下标位置上了，调用getEntryAfterMiss方法,该方法的签名如下:
```java
 private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e){
        Entry[]tab=table;
        int len=tab.length;

        while(e!=null){
        if(e.refersTo(key))
        return e;
        if(e.refersTo(null))
        expungeStaleEntry(i);
        else
        i=nextIndex(i,len);
        e=tab[i];
        }
        return null;
}
```
判断如果为空调用expungeStaleEntry方法清除无用的Entry对象，如果不为空就继续向前找得到下标，根据下标得到元素Entry，继续while循环判断得到元素Entry,判断是否相等，如果Entry为空了，说明没有相等的，返回null,get方法的逻辑还是比较简单的。

### 2.6 ThreadLocal的remove方法源码分析
该方法的签名如下：
```java
  private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.refersTo(key)) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
}
```
该方法逻辑同样很简单，获取到当前的ThreadLocal对象，计算出下标，如果得到元素Entry不为空，调用clear方法进行回收，然后又调用expungeStaleEntry方法清除无用的Entry对象。

至此ThreadLocal的主要方法set、get、remove方法都已经分析完成，希望对大家有所帮助。