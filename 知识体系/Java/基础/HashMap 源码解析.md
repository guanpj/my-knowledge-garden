---
title: HashMap 源码解析
comments: true
date created: 2023-03-21
date modified: 2023-03-22
id: home
layout: page
tags:
  - HashMap
  - 源码解析
  - Java
---
## 前言  
  
HashMap 是 Java 中最常用 K-V 容器，使用哈希值来确定元素存储的位置，HashMap 对 Entry 进行了扩展（Node），使其形成以链表或者树的形式存储在 HashMap 的容器里。  
  
在 Java 8 之前和之后，HashMap 的实现有较大的不同，因此对于 put 流程、扩容机制等主要过程分析将会采用两个版本进行对比。  
  
## 成员变量  
  
HashMap 成员变量和构造方法声明如下（Java 7 和 8 大致相同，以下为 Java 8 版本）：  
  
```java  
public class HashMap<K,V> extends AbstractMap<K,V>  
  
    implements Map<K,V>, Cloneable, Serializable {  

    // 初始容量 16  
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16  
  
    // 最大容量，该数组最大值为2^31一次方。  
    static final int MAXIMUM_CAPACITY = 1 << 30;  
  
    // 默认的加载因子，如果构造的时候不传则为 0.75  
    static final float DEFAULT_LOAD_FACTOR = 0.75f;  
  
    // @1.8：数组某个位置下的 node 链表转化为红黑树对该链表的最小长度要求  
    static final int TREEIFY_THRESHOLD = 8;  
  
    // @1.8：当一个反树化的阈值，当这个 node 长度减少到该值就会从树转化成链表  
    static final int UNTREEIFY_THRESHOLD = 6;  
  
    // @1.8：数组某个位置下的 node 链表转化为红黑树对元素个数的最小要求  
    static final int MIN_TREEIFY_CAPACITY = 64;  
  
    // 具体存放数据的数组  
    transient Node<K,V>[] table;  
  
    // entrySet，一个存放 k-v 缓冲区  
    transient Set<Map.Entry<K,V>> entrySet;  
  
    // 存放键值对的个数。  
    transient int size;  
  
    // 记录更改 map 结构次数(添加、删除、扩容？)  
    transient int modCount;  
  
    // 临界值，当实际大小(容量*填充因子)超过临界值时，会进行扩容  
  
    int threshold;  
  
    // 填充因子  
    final float loadFactor;  
  
    // 指定初始容量  
    public HashMap(int initialCapacity) {  
        this(initialCapacity, DEFAULT_LOAD_FACTOR);  
    }  

    // 默认构造函数  
    public HashMap() {  
        // 默认 threshold 在首次 put 时才复制，Java 7 则是调用  
        // this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR)  
        this.loadFactor = DEFAULT_LOAD_FACTOR;   
    }  

    // 包含另一个 Map  
    public HashMap(Map<? extends K, ? extends V> m) {  
        this.loadFactor = DEFAULT_LOAD_FACTOR;  
  
        putMapEntries(m, false);  
    }  

    // 指定初始容量和填充因子  
    public HashMap(int initialCapacity, float loadFactor) {  
        if (initialCapacity < 0) // 容量不能为负数  
  
            throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);  
  
        // 当容量大于 2^31 就取最大值 1<<30;   
  
        if (initialCapacity > MAXIMUM_CAPACITY)  
  
            initialCapacity = MAXIMUM_CAPACITY;  
  
        if (loadFactor <= 0 || Float.isNaN(loadFactor))  
  
            throw new IllegalArgumentException("Illegal load factor: "                + loadFactor);  
  
        this.loadFactor = loadFactor;  
  
        // tableSizeFor 保证了数组长度一定是 2 的幂次方，是大于等于    initialCapacity 最接近的值。  
        // 这里使用 threshold 暂时保存计算后的 initialCapacity 值  
        this.threshold = tableSizeFor(initialCapacity);  
  
    }  
    ...  
}  
```  
  
## 数据结构  
  
Java 7 采用数组 + 链表方式进行存储，元素类型为 Entry：  
  
```java  
static class Entry<K,V> implements Map.Entry<K,V> {  
  
    final K key;  
  
    V value;  
  
    Entry<K,V> next;  
  
    int hash;  

    ...  
}  
```  
  
元素的存储结构如下：  
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/HashMap/clipboard_20230322_035018.png)
Java 8 开始，采用数组 + 链表 + 红黑树方式进行存储，元素类型为 Node 和 TreeNode：  
  
```java  
static class Node<K,V> implements Map.Entry<K,V> {  
  
    final int hash;  
  
    final K key;  
  
    V value;  
  
    Node<K,V> next;  

    ...  
}  
  
  
  
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {  
  
    TreeNode<K,V> parent;  // red-black tree links  
  
    TreeNode<K,V> left;  
  
    TreeNode<K,V> right;  
  
    TreeNode<K,V> prev;    // needed to unlink next upon deletion  
  
    boolean red;  
    
    ...  
}  
```  
  
元素的存储结构如下：
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/HashMap/clipboard_20230322_041741.png)

## put 流程分析  
  
### Java 7 put 流程  
  
#### 代码分析  
  
Java 7 put 流程主要方法源码如下：  
  
```java  
public V put(K key, V value)  
    if (table == EMPTY_TABLE) {  
        // 初始化 table  
        inflateTable(threshold);  
    }  
  
    if (key == null) {  
        // 在 table[0] 处插入 key 为 null 元素并返回  
        return putForNullKey(value);  
    }  
  
    // 先进行一次 hash 计算     
    int hash = hash(key);  
  
    // 根据 hash 值计算 table 下标  
    int i = indexFor(hash, table.length);  
  
    // 遍历 table[i] 处的链表  
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {  
        Object k;  
        // hash 一样且 key 相等或者 equals 方法返回 true 才进行替换  
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {  
            V oldValue = e.value;  
  
            e.value = value;  
  
            e.recordAccess(this);  
  
            return oldValue;  
        }  
    }  
    // 出循环意味着 table[i] 这条链表没有此元素  
    // 更新 modCount  
    modCount++;  
    // 插入新元素  
    addEntry(hash, key, value, i);  
  
    return null;  
}  
  
private void inflateTable(int toSize) {  
    // 获取大于  
    int capacity = roundUpToPowerOf2(toSize);  
    // 重新计算阈值  
    threshold = (int) Math.min(capacity * loadFactor,   
            MAXIMUM_CAPACITY + 1);  
  
    // 创建数组  
    table = new Entry[capacity];  
  
    // 根据配置判断是否初始化 hashSeed  
    initHashSeedAsNeeded(capacity);  
}  
  
private V putForNullKey(V value) {  
    // 最多循环一次，因为这个位置最多只有一个元素  
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {  
        // key 为 null 直接替换  
        if (e.key == null) {  
            V oldValue = e.value;  
            e.value = value;  
            e.recordAccess(this);  
  
            return oldValue;  
        }  
    }  
  
    // 更新 modCount  
    modCount++;  
  
    // 在 table[0] 处插入元素  
    addEntry(0, null, value, 0);  
  
    return null;  
}  
  
void addEntry(int hash, K key, V value, int bucketIndex) {  
    // 元素数量达到临界值且 table[bucketIndex] 位置不为空才进行扩容  
    if ((size >= threshold) && (null != table[bucketIndex])) {  
        // 两倍容量  
        resize(2 * table.length);  
  
        // 重新计算 hash  
        hash = (null != key) ? hash(key) : 0;  
  
        // 重新确定数组下标  
        bucketIndex = indexFor(hash, table.length);  
    }
    // 创建并插入新元素  
    createEntry(hash, key, value, bucketIndex);  
}  
  
// 在链表头部插入新元素  
void createEntry(int hash, K key, V value, int bucketIndex) {  
    // 保存原头结点  
    Entry<K,V> e = table[bucketIndex];  
    // 创建新元素、把 next 指向头节点，并替换原来 bucketIndex 位置的链表  
    table[bucketIndex] = new Entry<>(hash, key, value, e);  
    // 元素数量++  
    size++；  
}  
```  
  
#### 流程图示  
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/HashMap/clipboard_20230322_040358.png)
### Java 8 put 流程  
  
#### 代码分析  
  
Java 8 put 流程主要方法源码如下：  
  
```java  
public V put(K key, V value) {  
      // onlyIfAbsent 默认为 false，即元素存在时进行替换  
    return putVal(hash(key), key, value, false, true);  
}  
  
final V putVal(int hash, K key, V value,   
          boolean onlyIfAbsent, boolean evict) {  
    Node<K,V>[] tab; Node<K,V> p; int n, i;  
  
    // table 未初始化或者长度为 0，进行扩容  
    if ((tab = table) == null || (n = tab.length) == 0)  
        n = (tab = resize()).length;  
  
    // (n - 1) & hash 确定元素存放位置，位置为空则直接放入该位置  
    if ((p = tab[i = (n - 1) & hash]) == null)  
        tab[i] = newNode(hash, key, value, null);  
  
    // 数组对应位置已经存在元素  
    else {  
        Node<K,V> e; K k;  
  
        // 比较数组中第一个元素  
        if (p.hash == hash &&  
            ((k = p.key) == key || (key != null && key.equals(k))))  
  
                // 将第一个元素赋值给 e  
                e = p;  
        // 是否为红黑树结点  
        else if (p instanceof TreeNode)  
            // 放入树中  
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab,   
                        hash, key, value);  
        // 链表结点  
        else {  
            // 遍历链表  
            for (int binCount = 0; ; ++binCount) {  
                // 到达链表的尾部，说明没有找到相等的 key  
                if ((e = p.next) == null) {  
                    // 在尾部插入新结点  
                    p.next = newNode(hash, key, value, null);  
  
                    // 判断结点数量是否达到阈值(TREEIFY_THRESHOLD 默认为 8)  
                    if (binCount >= TREEIFY_THRESHOLD - 1) {  
                        // 根据数组长度决定是否树化  
                        treeifyBin(tab, hash);  
                    }  
  
                    // 跳出循环  
                    break;  
                }  
  
                // 判断链表中结点的 key 值是否与插入的 key 相等  
                if (e.hash == hash &&  
                    ((k = e.key) == key ||   
                          (key != null && key.equals(k))))  
                    // key 相等，跳出循环，此时 e 就是目标结点  
                    break;  
  
                // 与前面的 e = p.next 组合遍历链表  
                p = e;  
            }  
        }  
  
        // 找到 key 值相等的目标结点  
        if (e != null) {  
  
            V oldValue = e.value;  
  
            // onlyIfAbsent 为 false 或者目标接点值为 null  
            if (!onlyIfAbsent || oldValue == null)  
  
                //用新值替换旧值  
                e.value = value;  
  
            // 空实现，用于访问后回调给子类，如 LinkedHashMap  
            afterNodeAccess(e);  
  
            // 返回旧值  
            return oldValue;  
        }  
    }  
  
    // 结构修改，更新 modCount  
    ++modCount;  
  
    // 实际大小大于阈值则扩容  
    if (++size > threshold)  
        resize();  
  
    // 插入后回调  
    afterNodeInsertion(evict);  
  
    return null;  
}  
```  
  
#### 流程图示  
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/HashMap/clipboard_20230322_040522.png)
## hash 计算与元素位置确定  
  
### Java 7  
  
```java  
int hash = hash(key);  
  
int i = indexFor(hash, table.length);  
  
final int hash(Object k) {  
  
    // 默认为 0，初始化方法见后文  
    int h = hashSeed;  
  
    // 如果 hashSeed 不为零且 key 是 String 类型  
    if (0 != h && k instanceof String) {  
        // 返回特定 hash 值  
        return sun.misc.Hashing.stringHash32((String) k);  
    }  
  
    h ^= k.hashCode();  
  
    // 多次异或  
    h ^= (h >>> 20) ^ (h >>> 12);  
  
    return h ^ (h >>> 7) ^ (h >>> 4);  
}  
  
static int indexFor(int h, int length) {  
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";  
    return h & (length-1);  
}  
```  
  
### Java 8  
  
```java  
public V put(K key, V value) {  
    return putVal(hash(key), key, value, false, true);  
}  

static final int hash(Object key) {  
    int h;  
  
    // 让高 16 位和低 16 位异或  
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  
}  

p = tab[index = (n - 1) & hash]；  
```  
  
## 扩容流程  
  
扩容过程涉及到 rehash、复制数据等操作，非常消耗性能。  
  
和扩容相关的全局变量及其含义：  
  
| 全局变量   | 含义                                                                                                                                                                                                                                                                                                                                                                                                      |  
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |  
| capacity   | table 的容量大小，默认为 16。需要注意的是 capacity 必须保证为 2 的 n 次方。                                                                                                                                                                                                                                                                                                                               |  
| size       | 存放键值对数量。                                                                                                                                                                                                                                                                                                                                                                                          |  
| threshold  | size 的临界值，当 size 大于等于 threshold 就必须进行扩容操作。threshold = (int) (capacity * loadFactor)                                                                                                                                                                                                                                                                                                   |  
| loadFactor | 填充因子，table 能够使用的比例。loadFactor 能够控制数组存放数据的疏密程度，loadFactor 越趋近于 1，那么数组中存放的数据(entry)也就越多，也就越密，也就是会让链表的长度增加，loadFactor 越小，也就是趋近于 0，数组中存放的数据(entry)也就越少，也就越稀疏。<br/>loadFactor 太大导致查找元素效率低，太小导致数组的利用率低，存放的数据会很分散。loadFactor 的默认值为 0.75f 是官方给出的一个比较好的临界值。 |  
  
### Java 7 扩容流程  
  
#### 代码分析  
  
Java 7 的 resize 方法相关代码：  
  
```java  
void resize(int newCapacity) {  
    Entry[] oldTable = table;  
  
    // 记录旧容量  
    int oldCapacity = oldTable.length;  
  
    // 如果容量已达到上限，则扩容阈值设置成不可能达到的最大值，即后续不再扩容  
    if (oldCapacity == MAXIMUM_CAPACITY) {  
        threshold = Integer.MAX_VALUE;  
  
        return;  
    }  
  
    // 根据新容量创建出新数组  
    Entry[] newTable = new Entry[newCapacity];  
  
    // 将旧数组的节点转移到新数组  
    transfer(newTable, initHashSeedAsNeeded(newCapacity));  
  
    // 新旧易主  
    table = newTable;  
  
    // 根据新容量重新确定新阈值  
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);  
}  
 
// 从配置中获取是否启用备用 hash，用于减少字符串 hash 冲突  
final boolean initHashSeedAsNeeded(int capacity) {  
    // 是否已经启用备用 hash  
    boolean currentAltHashing = hashSeed != 0;  
  
    // 虚拟机已经启动且数组容量大于 ALTERNATIVE_HASHING_THRESHOLD  
    boolean useAltHashing = sun.misc.VM.isBooted() &&  
            (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);  
  
    // 异或操作判断是否切换  
    boolean switching = currentAltHashing ^ useAltHashing;  
    if (switching) {  
        // useAltHashing 为 true 则 hashSeed 初始化为也给随机 hash 值  
        hashSeed = useAltHashing  
            ? sun.misc.Hashing.randomHashSeed(this)  
            : 0;  
    }  
    return switching;  
}  

void transfer(Entry[] newTable, boolean rehash) {  
    int newCapacity = newTable.length;  
  
    // 遍历旧数组  
    for (Entry<K,V> e : table) {  
        // 遍历数组上的链表  
        while(null != e) {  
            // 记录下一个位置  
            Entry<K,V> next = e.next;  
  
            // 判断是否重新计算 hash 值  
            if (rehash) {  
  
                e.hash = null == e.key ? 0 : hash(e.key);  
  
            }  
  
            // 根据新容量重新计算位置  
            int i = indexFor(e.hash, newCapacity);  
  
            // 按旧链表的正序遍历链表、在新链表的头部依次插入  
            // 因此扩容后可能出现逆序  
            e.next = newTable[i];  
  
            newTable[i] = e;  
  
            e = next;  
        }  
    }  
}  
```  
  
#### 扩容流程在多线程环境下的隐患  
  
在 resize 扩容过程中，在将旧数组上的数据转移到新数组上时，转移数据操作是按旧链表的正序遍历链表、在新链表的头部依次插入的。在多线程的环境下，由于这些操作不具有原子性和内存可见性，转移数据、扩容后，容易出现环形链表的情况。  
  
### Java 8 扩容流程  
  
#### 代码分析  
  
Java 8 中的 resize 和 treeifyBin 方法：  
  
```java  
final void treeifyBin(Node<K,V>[] tab, int hash) {  
    int n, index; Node<K,V> e;  
    // 如果数组为空或者数组长度小于 64，则进行扩容  
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)  
        resize();  
    // 根据 hash 获取数组下标，该位置有值再进行树化  
    else if ((e = tab[index = (n - 1) & hash]) != null) {  
        TreeNode<K,V> hd = null, tl = null;  
        // 遍历链表  
        do {  
            // Node 节点转换成 TreeNode 节点  
            TreeNode<K,V> p = replacementTreeNode(e, null);  
  
            if (tl == null)  
                hd = p;  
            else {  
                p.prev = tl;  
  
                tl.next = p;  
            }  
  
            tl = p;  
        } while ((e = e.next) != null);  
        
        if ((tab[index] = hd) != null)  
            hd.treeify(tab);  
    }  
}  

final Node<K,V>[] resize() {  
  
    Node<K,V>[] oldTab = table;  
  
    int oldCap = (oldTab == null) ? 0 : oldTab.length;  
  
    int oldThr = threshold;  
  
    int newCap, newThr = 0;  
  
    if (oldCap > 0) {  
        // 超过最大值后续不再扩容  
        if (oldCap >= MAXIMUM_CAPACITY) {  
            threshold = Integer.MAX_VALUE;  
            return oldTab;  
        }  
  
        // 否则扩充为原来的2倍  
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)  
            newThr = oldThr << 1; // double threshold  
    }  
  
    else if (oldThr > 0) // initial capacity was placed in threshold  
        newCap = oldThr;  
    else {  
        // signifies using defaults  
        newCap = DEFAULT_INITIAL_CAPACITY;  
  
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  
    }  
  
    // 计算新的 resize 上限  
  
    if (newThr == 0) {  
        float ft = (float)newCap * loadFactor;  
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);  
    }  
  
    threshold = newThr;  
  
    @SuppressWarnings({"rawtypes","unchecked"})  
  
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  
  
    table = newTab;  

    if (oldTab != null) {  
        // 旧数组迁移至新数组  
        for (int j = 0; j < oldCap; ++j) {  
            Node<K,V> e;  
            if ((e = oldTab[j]) != null) {  
                oldTab[j] = null;  
                if (e.next == null)  
                    newTab[e.hash & (newCap - 1)] = e;  
                else if (e instanceof TreeNode)  
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);  
                else {  
                    Node<K,V> loHead = null, loTail = null;  
                    Node<K,V> hiHead = null, hiTail = null;  
                    Node<K,V> next;  
                    do {  
                        next = e.next;  
                        // 原索引  
                        if ((e.hash & oldCap) == 0) {  
                            if (loTail == null)  
                                loHead = e;  
                            else  
                                loTail.next = e;  
                            loTail = e;  
                        }  
  
                        // 原索引+oldCap  
                        else {  
                            if (hiTail == null)  
                                hiHead = e;  
                            else  
                                hiTail.next = e;  
                            hiTail = e;  
                        }  
                    } while ((e = next) != null);  
  
                    // 原索引元素放到新数组中  
                    if (loTail != null) {  
                        loTail.next = null;  
                        newTab[j] = loHead;  
                    }  
  
                    // 原索引 +oldCap 元素放到新数组中  
                    if (hiTail != null) {  
                        hiTail.next = null;  
                        newTab[j + oldCap] = hiHead;  
                    }  
                }  
            }  
        }  
    }  
    return newTab;  
}  
```  
  
resize 方法中的前半段，关于 newCap 和 newThr 的计算过程，简化后如下：  
  
```java  
if (oldCap > 0) {  
    // 嵌套条件分支  
    if (oldCap >= MAXIMUM_CAPACITY) {...}  
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&  
                 oldCap >= DEFAULT_INITIAL_CAPACITY) {...}  
}   
else if (oldThr > 0) {...}  
  
else {...}  
```  
  
这些判断分别对应以下几种条件：  
  
| 条件                       | 覆盖情况                            | 备注                                                                                                                         |  
| -------------------------- | ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |  
| oldCap > 0                 | 桶数组 table 已经被初始化           |                                                                                                                              |  
| oldThr > 0                 | threshold > 0，且桶数组未被初始化   | 调用 HashMap(int) 和 HashMap(int, float) 构造方法时会产生这种情况，此种情况下 newCap = oldThr，newThr 在第二个条件分支中算出 |  
| oldCap == 0 && oldThr == 0 | 桶数组未被初始化，且 threshold 为 0 | 调用 HashMap() 构造方法会产生这种情况。                                                                                      |  
  
oldCap > 0 时表示 table 数组已经被初始化过，这时需要再次计算容量和阈值：  
  
| 条件                        | 覆盖情况                                      | 备注                                                      |  
| --------------------------- | --------------------------------------------- | --------------------------------------------------------- |  
| oldCap >= 230               | 桶数组容量大于或等于最大桶容量 230            | 后续不再扩容                                              |  
| newCap < 230 && oldCap > 16 | 新桶数组容量小于最大值，且旧桶数组容量大于 16 | 该种情况下新阈值 newThr = oldThr << 1，移位可能会导致溢出 |  
  
resize 方法后续过程中可以看出，Java 8 转移数据操作是按旧链表的正序遍历链表、在新链表的尾部依次插入的，所以不会出现链表 逆序、倒置的情况，故不容易出现环形链表的情况。但仍然还是线程不安全，因为没有加同步锁保护。  
  
### 扩容流程对比  
![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/HashMap/clipboard_20230322_040805.png)
## get 流程分析  
  
### Java 7
  
```java  
public V get(Object key) {  
    if (key == null)  
        return getForNullKey();  
    Entry<K,V> entry = getEntry(key);  
    return null == entry ? null : entry.getValue();  
}  
  
private V getForNullKey() {  
    if (size == 0) {  
        return null;  
    }  
  
    // 下标为 0 处获取 key 为 null 的元素  
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {  
        if (e.key == null)  
            return e.value;  
    }  
    return null;  
}  

final Entry<K,V> getEntry(Object key) {  
    if (size == 0) {  
        return null;  
    }  
  
    int hash = (key == null) ? 0 : hash(key);  
    for (Entry<K,V> e = table[indexFor(hash, table.length)];  
             e != null; e = e.next) {  
        Object k;  
        if (e.hash == hash &&  
            ((k = e.key) == key || (key != null && key.equals(k))))  
            return e;  
    }  
    return null;  
}  
```  
  
### Java 8
  
```java  
public V get(Object key) {  
    Node<K,V> e;  
    return (e = getNode(hash(key), key)) == null ? null : e.value;  
}  

final Node<K,V> getNode(int hash, Object key) {  
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;  
    if ((tab = table) != null && (n = tab.length) > 0 &&  
        (first = tab[(n - 1) & hash]) != null) {  
        // 先判断 tab[index] 中的第一个元素  
        if (first.hash == hash && // always check first node  
            ((k = first.key) == key || (key != null && key.equals(k))))  
            return first;  
        if ((e = first.next) != null) {  
            // 结点为红黑树则使用 TreeNode 的方法获取  
            if (first instanceof TreeNode)  
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);  
            do { //否则遍历链表  
                if (e.hash == hash &&  
                    ((k = e.key) == key || (key != null && key.equals(k))))  
                    return e;  
            } while ((e = e.next) != null);  
        }  
    }  
    return null;  
}  
```  
  
遍历方式  
  
```java  
Map<String, Integer> map = new HashMap<String, Integer>() {{  
    put("a", 10);  
    put("b", 20);  
}};  
  
// 方式一：迭代 entrySet  
  
for (Map.Entry<String, Integer> entry : map.entrySet()) {  
    String key = entry.getKey();  
    int value = entry.getValue();  
}  

// 方式二：单独迭代 keySet 或 values  
// 迭代键  
for (String key : map.keySet()) {  
    System.out.println("Key = " + key);  
}  
  
// 迭代值  
for (Integer value : map.values()) {  
    System.out.println("Value = " + value);  
}  

// 方式三：使用 iterator  
  
Iterator<Map.Entry<String, Integer>> entries =         
          map.entrySet().iterator();  
  
while (entries.hasNext()) {  
    Map.Entry<String, Integer> entry = entries.next();  
    String key = entry.getKey();  
    int value = entry.getValue();  
}  

// 方式四：Lambda 表达式  
map.forEach((k, v) -> System.out.println("key: " + k + " value:" + v));  
```  
  
## 面试题  
  
### 为什么 HashMap 数组长度为 2 的幂次方？  
  
如果数组中 hash 冲突太过频繁，某些位置的元素形成过长的链表，就会导致数据存取效率过低，因此要使元素均匀地分布在数组中，hash 碰撞就不能太频繁。而 hash 值范围为 -2147483648 到 2147483647，如果用如此长的数组来进行存放加上合理 hash 映射确实可以使 hash 碰撞降到很低的水平，但这明显是不现实的。  
  
因此这个 hash 值是不能直接使用的，首先需要进行二次 hash 使得结果更加随机，这时理论上可以使用 hash % length 得到一个不小于数组长度 length 的 index 值，这样不但能避免产生数组越界，并且可以使元素均匀分布，事实上很多 hash 算法都是采用该方法。但是在计算机中，% 取模运算比位 & 运算的效率要低得多，而当 length 为 2 的幂次方时，hash % length 刚好等于 hash & (length - 1) ，从而能够将 % 运算转换成 & 运算，更加快速地得到 index 值。  
  
因此采用 2 的幂次方作为数组的长度的好处是：使元素均匀分部以降低 hash 冲突的基础上，大大加快了计算元素所在数组位置的速度。  
  
### hash 冲突有哪些解决方法？HashMap 是怎样解决的？  
  
1. 二次 hash，通过高 16 位与低 16 位异或运算，使得结果更加随机；  
2. 拉链地址法，将 hash 值相同的元素串成一个链表或者转为红黑树。  
  
### HashMap 会造成哪些安全问题？怎么解决？  
  
如前面文章所述，在 Java 7 中，HashMap 在扩容的时候是通过遍历旧数组，然后在新数组中使用头插法进行转移元素的。这在单线程环境中是没有问题的，但是到了多线程环境下，由于 JMM 的特性，会以一定的概率形成环形链表的情况。在 Java 8 中这个问题通过使用尾插法得到解决，但是多线程下很多操作仍然会导致线程安全问题，比如多个线程 put 后某些元素丢失等。因此多线程环境下要保证线程安全，可以使用 ConcurentHashMap 代替 HashMap。  
  
### 使用对象作为 HashMap 的 key 应该注意什么？  
  
应当重写对象的 hashCode() 和 equals() 方法。如前面代码所示，HashMap 在 put 一个元素的时候，会调用此元素的 key 值的 hashCode 方法确定元素存放位置，并且会调用 hashCode 和 equals 方法判断元素是否相等。因此如果没有重写这两个方法，或者方法重写的时候没有遵守规则，HashMap 通过 put 存入一组元素后，再通过此元素的 key 值去 get 对象的时候就有可能出现跟预期结果不一致的情况。  
  
在重写 equals 方法的时候，需要遵守下面的通用约定：  
  
- <strong>自反性</strong>：对于任何非空引用 x，x.equals(x) 必须返回 true；  
- <strong>对称性</strong>：对于任何非空引用 x 和 y，如果且仅当 y.equals(x) 返回 true 时 x.equals(y) 必须返回 true；  
- <strong>传递性</strong>：对于任何非空引用 x、y、z，如果 x.equals(y) 返回 true，y.equals(z) 返回 true，则 x.equals(z) 必须返回 true；  
- <strong>一致性</strong>：对于任何非空引用 x 和 y，如果在 equals 比较中使用的信息没有修改，则 x.equals(y) 的多次调用必须始终返回 true 或始终返回 false；  
- <strong>非空性</strong>：对于任何非空引用 x，x.equals(null) 必须返回 false。  
  
<strong>重写 hashCode 方法的大致方式（非强制）：</strong>  
  
1. 把某个非零常数值，比如说 31（最好是素数，考虑到 HashMap 源码中的异或操作），保存在一个叫 result 的 int 类型的变量中。  
2. 对于对象中每一个关键域 f（值 equals 方法中考虑的每一个域），完成以下步骤：  
3. 为该域计算 int 类型的散列码 c:  
  
```  
1. 如果该域是 boolean 类型，则计算 (f?0:1)  
  
2. 如果该域是 byte、char、short 或者 int 类型，则计算 (int)f  
  
3. 如果该域是 float 类型，则计算 Float.floatToIntBits(f)  
  
4. 如果该域是 long 类型，则计算 (int)(f ^ (f>>>32))  
  
5. 如果该域是double类型，则计算 Double.doubleToLongBits(f) 得到一个 long 类型的值，然后按照步骤 4，对该 long 型值计算散列值  
  
6. 如果该域是一个对象引用，并且该类的 equals 方法通过递归调用 equals 的方式来比较这个域，则同样对这个对象递归调用 hashCode 方法。  
  
7. 如果该域是一个数组，则把每一个元素当做单独的域来处理。也就是说，递归地应用上述规则，对每个重要的元素计算一个散列码，然后根据步骤下面的做法把这些散列值组合起来。  
```  
  
2. 按照下面的公式，把步骤 1 中计算得到的散列码 c 组合到 result 中：result = 31 x result+c。  
3. 返回 result。  
4. 写完 hashCode 方法之后，确认是否相等的实例具有相等的散列码。如果不是的话，找出原因，并修改。  
  
样例：  
  
```java  
public class Student {  
  
    private String name;  
  
    private int age;  
  
    private Grades grades;  
  
    public Student(String name, int age, Grades grades) {  
        this.name = name;  
        this.age = age;  
        this.grades = grades;  
    }  

    @Override  
    public int hashCode() {  
        final int prime = 31;  
        int result = 1;  
        
        result = prime * result + age;  
        result = prime * result +  
                ((name == null) ? 0 : name.hashCode());  
        result = prime * result +  
                (grades == null ? 0 : grades.hashCode());  
  
        return result;  
  
    }  
  
    @Override  
    public boolean equals(Object obj) {  
        if (this == obj) return true;  
  
        if (obj == null || getClass() != obj.getClass()) return false;  
  
        Student other = (Student) obj;  
  
        if (age != other.age) return false;  
  
        if (name != null ? !name.equals(other.name) :   
              other.name != null) return false;  
  
        if (grades != null ? !grades.equals(other.grades) :           
              other.grades != null) return false;  
  
        return true;  
    }  
}  
```