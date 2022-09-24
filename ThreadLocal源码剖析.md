# 简介

ThreadLocalMap Entry<ThreadLocal<?>, Object>继承了WeakReference是弱引用，如果没有接收它的对象将它升级为强引用，会在下一次gc发生时回收掉，此时key为null，而value任然存在，可能会引起内存泄露

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

ThreadLocalMap过期key的两种清理方式：**探测式清理**expungeStaleEntry、**启发式清理**cleanSomeSlots

每当创建一个ThreadLocal对象，这个ThreadLocal.nextHashCode 这个值就会增长 `0x61c88647(HASH_INCREMENT)` 

```java
private static final int HASH_INCREMENT = 0x61c88647;
```

这个值很特殊，它是斐波那契数 也叫 黄金分割数。hash增量为 这个数字，带来的好处就是 hash 分布非常均匀。

# set方法

1. 从当前hash下标开始往后遍历
      - k=key 找到key相同的元素，直接更新value并return
      - k=null 找到一个过期元素 执行replaceStaleEntry()方法

2. 步骤1遍历结束，如果没有找到当前元素也没有发现过期元素，直接创建一个新的Entry放进去，并++size
   - 进行启发式清理(cleanSomeSlots)，如果未清理任何数据，且当前散列数组中Entry的数量已经达到了列表的扩容阈值(len*2/3)，就开始执行rehash()方法
   - 进行探测式清理，如清理后size还是大于等于阈值的3/4，则执行resize() 扩容至原来的两倍]


```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;								
    int len = tab.length;								
    int i = key.threadLocalHashCode & (len-1);								
								
    for (Entry e = tab[i];								
         e != null;								
         e = tab[i = nextIndex(i, len)]) {								
        ThreadLocal<?> k = e.get    ();								
 
        if (k == key) {
            e.value = value;
            return;
        }
 
        if (k == null) {
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

```java
private void rehash() {
    expungeStaleEntries();
 
    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}
```

```java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

```java
/**
 * 将容量扩大至当前的2倍
 */
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
 
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
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
```

## replaceStaleEntry方法

1. 往前遍历(第一个for循环)，找到过期元素将slotToExpunge更新为其下标，并return
2. 往后遍历(第二个for循环)
   - k=key 将value更新，将当前元素与当前过期元素位置替换，然后调用cleanSomeSlots(expungeStaleEntry(slotToExpunge), len)删除过期元素，return
   - k=null && slotToExpunge == staleSlot 当前元素为过期元素，且步骤①没有找到其他过期元素时，将slotToExpunge更新为当前下标
3. 在步骤2中没有找到key相等的元素的话会走到这里，new Entry替换到当前位置过期的元素
4. 在步骤1中如果扫描到了其他过期元素，调用cleanSomeSlots(expungeStaleEntry(slotToExpunge), len)删除过期元素

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
        if (e.get() == null)
            slotToExpunge = i;
 
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
 
        if (k == key) {
            e.value = value;
 
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;
 
            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }
 
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }
 
    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
 
    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

## expungeStaleEntry 探测式清理

1. 删除当前元素
2. 继续往后遍历
   - k=null 找到key为空的过期元素，将它删除
   - k!=null 将因hash碰撞后移的非空元素重新放置(例：删掉了下标为8的元素之后，将下标为9的元素(hash值不为9)重新放置到8的位置)

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
 
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

## cleanSomeSlots 启发式清理

往后遍历

- 如果找到key为null的过期元素， 则调用expungeStaleEntry探测式清理

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

# get方法

获取当前线程的ThreadLocalMap
- 取到map不为空，获取当前entry
- 取到map或者entry为空，执行setInitialValue方法

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

## getEntry

计算hash值
- key相同，直接返回
- key不同，执行getEntryAfterMiss方法

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

## getEntryAfterMiss

往后遍历

- key相同，直接返回
- k=null，触发探测式清理操作，清理后过期数据会被回收，正常数据会往前移
- 遍历下一个entry

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
 
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

## setInitialValue

- map!=null，直接执行set操作
- map为空，new一个map

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
    if (this instanceof TerminatingThreadLocal) {
        TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
    }
    return value;
}
```

```java
protected T initialValue() {
    return null;
}
```

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

