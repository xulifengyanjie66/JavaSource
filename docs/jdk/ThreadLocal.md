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
