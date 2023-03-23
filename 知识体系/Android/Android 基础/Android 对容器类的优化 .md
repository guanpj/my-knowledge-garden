# Android 对容器类的优化

在之前 [HashMap 的源码解析的文章](https://ywue4d2ujm.feishu.cn/docs/doccnWxrCbptIcob6fIMUWVeWfc?app_id=11#S0X9Es)中可知，HashMap 的一级存储结构是一个初始容量为 16 的数组， 所以当我们创建出一个 HashMap 对象时，即使里面没有任何元素，也要分别一块内存空间给它。而且，在不断的向 HashMap 里 put 数据的过程中，当数据量达到阈值（容量*加载因子，加载因子默认为 0.75）时，将会触发 HashMap 扩容流程，扩大后新的容量一定是原来的 2 倍。

假如我们有几十万、几百万条数据，那么 HashMap 要存储完这些数据将要不断的扩容，而且在此过程中也需要不断的做 hash 运算，这将对我们的内存空间造成很大消耗和浪费，再加上 HashMap 获取数据是通过遍历 Entry[] 数组来得到对应的元素，在数据量很大时候会比较慢。

所以对于运行在移动设备上的 Android 系统来说，HashMap 的使用会造成比较大的负担。因此在

android.util 包中，提供了几个容器类，在某些情况下可以取代 HashMap 以提升性能。

# SparseArray

相比于比 HashMap ，SparseArray 更省内存，并且在某些条件下性能更好，主要是因为它避免了对 key 的自动装箱（int 转为 Integer 类型）。与普通的对象数组不同，它的索引可以包含间隙，因此得名 SparseArray（稀疏数组）。它的内部采用数组来存储 key，并且通过二分查找定位到目标 key，因此在数据量达到数百个时，效率将会降低至少 50% 之于 HashMap。

为了提高性能，在删除某项数据时，SparseArray 并不会马上删除 value 中的内容并且压缩整理 key 数组，而是会把将要删除的数据标记成 <strong>DELETE</strong>，后续可以重新用于相同 key 值的数据或者在 gc 操作的时候进行删除。SparseArray 在进行扩容前或者调用 size、indexOfKey 等方法时必须进行 gc 操作。

```java
public class SparseArray<E> implements Cloneable {
    private static final Object DELETED = new Object ();
    private boolean mGarbage = false;
    private int [] mKeys;
    private Object [] mValues;

    public SparseArray () {
        this(10);
    }

    public SparseArray (int initialCapacity) {
        if (initialCapacity == 0) {
            mKeys = EmptyArray.INT;
            mValues = EmptyArray.OBJECT;
        } else {
            mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
            mKeys = new int [mValues.length];
        }
        mSize = 0;
    }
    
    ...
}
```

它内部则是通过两个数组来进行数据存储的，一个存储 key，另外一个存储 value。

## 添加数据

```typescript
public void put(int key, E value) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        mValues[i] = value;
    } else {
        i = ~i;

        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }

        if (mGarbage && mSize >= mKeys.length) {
            gc();

            // Search again because indices may have changed.
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }

        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```

## 删除数据

```java
public void delete(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}
```

## 获取数据

```java
public E get(int key) {
    return get(key, null);
}

public E get(int key, E valueIfKeyNotFound) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i < 0 || mValues[i] == DELETED) {
        return valueIfKeyNotFound;
    } else {
        return (E) mValues[i];
    }
}
```

## 特有方法

```java
public int keyAt(int index) {
    if (index >= mSize && UtilConfig.sThrowExceptionForUpperArrayOutOfBounds) {
        // The array might be slightly bigger than mSize, in which case, indexing won't fail.
        // Check if exception should be thrown outside of the critical path.
        throw new ArrayIndexOutOfBoundsException(index);
    }
    if (mGarbage) {
        gc();
    }

    return mKeys[index];
}


public E valueAt(int index) {
    if (index >= mSize && UtilConfig.sThrowExceptionForUpperArrayOutOfBounds) {
        // The array might be slightly bigger than mSize, in which case, indexing won't fail.
        // Check if exception should be thrown outside of the critical path.
        throw new ArrayIndexOutOfBoundsException(index);
    }
    if (mGarbage) {
        gc();
    }

    return (E) mValues[index];
}
```

## 垃圾回收

```java
private void gc() {
    // Log.e("SparseArray", "gc start with " + mSize);

    int n = mSize;
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;

    for (int i = 0; i < n; i++) {
        Object val = values[i];

        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                values[i] = null;
            }

            o++;
        }
    }

    mGarbage = false;
    mSize = o;

    // Log.e("SparseArray", "gc end with " + mSize);
}
```

## 小结

优点：

- 避免自动装箱，直接使用数组存储 int 类型的 key
- 二分查找加快定位 key
- 删除元素先进行标记，可以重复使用，gc 的时候再进行删除

应用场景：

- key 为 int 类型
- 数据量不大，最好在千级以内

# ArrayMap

ArrayMap 在设计上比 HashMap 更多的考虑了内存的优化，可以理解为以时间换空间的一种优化。它使用了两个数组来存储数据——一个整型数组存储键的 hash 值，另一个对象数组存储键/值对。这样既能避免为每个存入 Map 中的键创建额外的对象，又能更积极的控制这些数据的长度的增加。因为增加长度只需要拷贝数组中的键，而不是重新构建一个哈希表。

它和 SparseArray 一样，也会对 key 使用二分法进行从小到大排序，在添加、删除、查找数据的时候都是先使用二分查找法得到 key 相应的 index，然后通过 index 来进行添加、查找、删除等操作。所以，应用场景和 SparseArray 的一样，如果在数据量比较大的情况下，那么它的性能将退化至少 50 %。

```java
public final class ArrayMap<K, V> implements Map<K, V> {
    ...

    int[] mHashes;
    @UnsupportedAppUsage(maxTargetSdk = 28) // Storage is an implementation detail. Use public key/value API.
    Object[] mArray;
    @UnsupportedAppUsage(maxTargetSdk = 28) // Use size()
    int mSize;
    
    ...

    public ArrayMap() {
        this(0, false);
    }

    public ArrayMap(int capacity) {
        this(capacity, false);
    }

    public ArrayMap(int capacity, boolean identityHashCode) {
        mIdentityHashCode = identityHashCode;

        // If this is immutable, use the sentinal EMPTY_IMMUTABLE_INTS
        // instance instead of the usual EmptyArray.INT. The reference
        // is checked later to see if the array is allowed to grow.
        if (capacity < 0) {
            mHashes = EMPTY_IMMUTABLE_INTS;
            mArray = EmptyArray.OBJECT;
        } else if (capacity == 0) {
            mHashes = EmptyArray.INT;
            mArray = EmptyArray.OBJECT;
        } else {
            allocArrays(capacity);
        }
        mSize = 0;
    }

    public ArrayMap(ArrayMap<K, V> map) {
        this();
        if (map != null) {
            putAll(map);
        }
    }
    
    ...
}
```

ArrayMap 数据结构：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Android-Container/clipboard_20230323_042030.png)

如上图所示，在 ArrayMap 内部有两个比较重要的数组，一个是 mHashes，另一个是 mArray。

- mHashes 用来存放 key 的 hash 值
- mArray 用来存储 key 与 value 的值，它是一个 Object 数组

其中这两个数组的索引对应关系是

```java
mHashes[index] = hash;
mArray[index<<1] = key;  //等同于 mArray[index * 2] = key;
mArray[(index<<1)+1] = value; //等同于 mArray[index * 2 + 1] = value;
```

## 确定元素位置

```java
int indexOf(Object key, int hash) {
    final int N = mSize;
    //快速判断是ArrayMap是否为空,如果符合情况快速跳出
    if (N == 0) {
        return ~0;
    }
    //二分查找确定索引值
    int index = ContainerHelpers.binarySearch(mHashes, N, hash);

    // 如果未找到，返回一个index值，可能为后续可能的插入数据使用。
    if (index < 0) {
        return index;
    }

    // 如果确定不仅hashcode相同，也是同一个key，返回找到的索引值。
    if (key.equals(mArray[index<<1])) {
        return index;
    }

    // 如果key的hashcode相同，但不是同一对象，从索引之后再次找
    int end;
    for (end = index + 1; end < N && mHashes[end] == hash; end++) {
        if (key.equals(mArray[end << 1])) return end;
    }

    // 如果key的hashcode相同，但不是同一对象，从索引之前再次找
    for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
        if (key.equals(mArray[i << 1])) return i;
    }
    //返回负值，既可以用来表示无法找到匹配的key，也可以用来为后续的插入数据所用。
    // Key not found -- return negative value indicating where a
    // new entry for this key should go.  We use the end of the
    // hash chain to reduce the number of array entries that will
    // need to be copied when inserting.
    return ~end;
}
```

## 添加数据

键被插入到 objects 的下一个空闲位置。值对象被插入到 mArray 的与对应键相邻的位置。计算出的键的 hashCode 会被插入到 mHashes 数组的下一个空闲位置。

```java
public V put(K key, V value) {
    final int osize = mSize;
    final int hash;
    int index;
    if (key == null) {
        hash = 0;
        index = indexOfNull();
    } else {
        hash = mIdentityHashCode ? System.identityHashCode(key) : key.hashCode();
        index = indexOf(key, hash);
    }
    if (index >= 0) {
        index = (index<<1) + 1;
        final V old = (V)mArray[index];
        mArray[index] = value;
        return old;
    }

    index = ~index;
    if (osize >= mHashes.length) {
        final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

        if (DEBUG) Log.d(TAG, "put: grow from " + mHashes.length + " to " + n);

        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        allocArrays(n);

        if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
            throw new ConcurrentModificationException();
        }

        if (mHashes.length > 0) {
            if (DEBUG) Log.d(TAG, "put: copy 0-" + osize + " to 0");
            System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
            System.arraycopy(oarray, 0, mArray, 0, oarray.length);
        }

        freeArrays(ohashes, oarray, osize);
    }

    if (index < osize) {
        if (DEBUG) Log.d(TAG, "put: move " + index + "-" + (osize-index)
                + " to " + (index+1));
        System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
        System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
    }

    if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
        if (osize != mSize || index >= mHashes.length) {
            throw new ConcurrentModificationException();
        }
    }
    mHashes[index] = hash;
    mArray[index<<1] = key;
    mArray[(index<<1)+1] = value;
    mSize++;
    return null;
}
```

扩容逻辑：

- 首先数组的容量会扩充到 BASE_SIZE
- 如果 BASE_SIZE 无法容纳，则扩大到 2 * BASE_SIZE
- 如果 2 * BASE_SIZE 仍然无法容纳，则每次扩容为当前容量的 1.5 倍。

## 删除数据

```java
public V remove(Object key) {
    final int index = indexOfKey(key);
    if (index >= 0) {
        return removeAt(index);
    }

    return null;
}

public V removeAt(int index) {
    if (index >= mSize && UtilConfig.sThrowExceptionForUpperArrayOutOfBounds) {
        // The array might be slightly bigger than mSize, in which case, indexing won't fail.
        // Check if exception should be thrown outside of the critical path.
        throw new ArrayIndexOutOfBoundsException(index);
    }

    final Object old = mArray[(index << 1) + 1];
    final int osize = mSize;
    final int nsize;
    if (osize <= 1) {
        // Now empty.
        if (DEBUG) Log.d(TAG, "remove: shrink from " + mHashes.length + " to 0");
        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
        freeArrays(ohashes, oarray, osize);
        nsize = 0;
    } else {
        nsize = osize - 1;
        if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {
            // Shrunk enough to reduce size of arrays.  We don't allow it to
            // shrink smaller than (BASE_SIZE*2) to avoid flapping between
            // that and BASE_SIZE.
            final int n = osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2);

            if (DEBUG) Log.d(TAG, "remove: shrink from " + mHashes.length + " to " + n);

            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
            allocArrays(n);

            if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                throw new ConcurrentModificationException();
            }

            if (index > 0) {
                if (DEBUG) Log.d(TAG, "remove: copy from 0-" + index + " to 0");
                System.arraycopy(ohashes, 0, mHashes, 0, index);
                System.arraycopy(oarray, 0, mArray, 0, index << 1);
            }
            if (index < nsize) {
                if (DEBUG) Log.d(TAG, "remove: copy from " + (index+1) + "-" + nsize
                        + " to " + index);
                System.arraycopy(ohashes, index + 1, mHashes, index, nsize - index);
                System.arraycopy(oarray, (index + 1) << 1, mArray, index << 1,
                        (nsize - index) << 1);
            }
        } else {
            if (index < nsize) {
                if (DEBUG) Log.d(TAG, "remove: move " + (index+1) + "-" + nsize
                        + " to " + index);
                System.arraycopy(mHashes, index + 1, mHashes, index, nsize - index);
                System.arraycopy(mArray, (index + 1) << 1, mArray, index << 1,
                        (nsize - index) << 1);
            }
            mArray[nsize << 1] = null;
            mArray[(nsize << 1) + 1] = null;
        }
    }
    if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
        throw new ConcurrentModificationException();
    }
    mSize = nsize;
    return (V)old;
}
```

- 如果当前 ArrayMap 只有一项数据，则删除操作将 mHashes，mArray 置为空数组，mSize 置为 0
- 如果当前 ArrayMap 容量过大（大于 BASE_SIZE*2）并且持有的数据量过小（不足 1/3）则降低 ArrayMap 容量，减少内存占用
- 如果不符合上面的情况，则从 mHashes 删除对应的值，将 mArray 中对应的索引置为 null

## 获取数据

```java
public V get(Object key) {
    final int index = indexOfKey(key);
    return index >= 0 ? (V)mArray[(index<<1)+1] : null;
}

public int indexOfKey(Object key) {
    return key == null ? indexOfNull()
            : indexOf(key, mIdentityHashCode ? System.identityHashCode(key) : key.hashCode());
}

int indexOf(Object key, int hash) {
    final int N = mSize;

    // Important fast case: if nothing is in here, nothing to look for.
    if (N == 0) {
        return ~0;
    }

    int index = binarySearchHashes(mHashes, N, hash);

    // If the hash code wasn't found, then we have no entry for this key.
    if (index < 0) {
        return index;
    }

    // If the key at the returned index matches, that's what we want.
    if (key.equals(mArray[index<<1])) {
        return index;
    }

    // Search for a matching key after the index.
    int end;
    for (end = index + 1; end < N && mHashes[end] == hash; end++) {
        if (key.equals(mArray[end << 1])) return end;
    }

    // Search for a matching key before the index.
    for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
        if (key.equals(mArray[i << 1])) return i;
    }

    // Key not found -- return negative value indicating where a
    // new entry for this key should go.  We use the end of the
    // hash chain to reduce the number of array entries that will
    // need to be copied when inserting.
    return ~end;
}
```

## 特有方法

```typescript
public K keyAt(int index) {
    if (index >= mSize && UtilConfig.sThrowExceptionForUpperArrayOutOfBounds) {
        // The array might be slightly bigger than mSize, in which case, indexing won't fail.
        // Check if exception should be thrown outside of the critical path.
        throw new ArrayIndexOutOfBoundsException(index);
    }
    return (K)mArray[index << 1];
}


public V valueAt(int index) {
    if (index >= mSize && UtilConfig.sThrowExceptionForUpperArrayOutOfBounds) {
        // The array might be slightly bigger than mSize, in which case, indexing won't fail.
        // Check if exception should be thrown outside of the critical path.
        throw new ArrayIndexOutOfBoundsException(index);
    }
    return (V)mArray[(index << 1) + 1];
}
```

## 缓存优化

ArrayMap 的容量发生变化，正如前面介绍的，有这两种情况：

- put 方法增加数据，扩大容量
- remove 方法删除数据，减小容量

在这个过程中，会频繁出现多个容量为 BASE_SIZE 和 2 * BASE_SIZE 的 int 数组和 Object 数组。ArrayMap 设计者为了避免创建不必要的对象，减少 GC 的压力。采用了类似[对象池](https://droidyue.com/blog/2016/12/12/dive-into-object-pool/)的优化设计。

这其中涉及到几个元素：

- BASE_SIZE 值为 4，与 ArrayMap 容量有密切关系。
- mBaseCache 用来缓存容量为 BASE_SIZE 的 int 数组和 Object 数组
- mBaseCacheSize mBaseCache 缓存的数量，避免无限缓存
- mTwiceBaseCache 用来缓存容量为 BASE_SIZE * 2 的 int 数组和 Object 数组
- mTwiceBaseCacheSize mTwiceBaseCache 缓存的数量，避免无限缓存
- CACHE_SIZE 值为 10，用来控制 mBaseCache 与 mTwiceBaseCache 缓存的大小

这其中：

- mBaseCache 的第一个元素保存下一个 m BaseCache，第二个元素保存 mHashes 数组
- mTwiceBaseCache 和 mBaseCache 一样，只是对应的数组容量不同

具体的缓存数组逻辑的代码：

```java
private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
    if (hashes.length == (BASE_SIZE*2)) {
        synchronized (ArrayMap.class) {
            if (mTwiceBaseCacheSize < CACHE_SIZE) {
                array[0] = mTwiceBaseCache;
                array[1] = hashes;
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                mTwiceBaseCache = array;
                mTwiceBaseCacheSize++;
                if (DEBUG) Log.d(TAG, "Storing 2x cache " + array
                        + " now have " + mTwiceBaseCacheSize + " entries");
            }
        }
    } else if (hashes.length == BASE_SIZE) {
        synchronized (ArrayMap.class) {
            if (mBaseCacheSize < CACHE_SIZE) {
                array[0] = mBaseCache;
                array[1] = hashes;
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                mBaseCache = array;
                mBaseCacheSize++;
                if (DEBUG) Log.d(TAG, "Storing 1x cache " + array
                        + " now have " + mBaseCacheSize + " entries");
            }
        }
    }

```

具体的利用缓存数组的代码：

```java
private void allocArrays(final int size) {
    if (mHashes == EMPTY_IMMUTABLE_INTS) {
        throw new UnsupportedOperationException("ArrayMap is immutable");
    }
    if (size == (BASE_SIZE*2)) {
        synchronized (ArrayMap.class) {
            if (mTwiceBaseCache != null) {
                final Object[] array = mTwiceBaseCache;
                mArray = array;
                mTwiceBaseCache = (Object[])array[0];
                mHashes = (int[])array[1];
                array[0] = array[1] = null;
                mTwiceBaseCacheSize--;
                if (DEBUG) Log.d(TAG, "Retrieving 2x cache " + mHashes
                        + " now have " + mTwiceBaseCacheSize + " entries");
                return;
            }
        }
    } else if (size == BASE_SIZE) {
        synchronized (ArrayMap.class) {
            if (mBaseCache != null) {
                final Object[] array = mBaseCache;
                mArray = array;
                mBaseCache = (Object[])array[0];
                mHashes = (int[])array[1];
                array[0] = array[1] = null;
                mBaseCacheSize--;
                if (DEBUG) Log.d(TAG, "Retrieving 1x cache " + mHashes
                        + " now have " + mBaseCacheSize + " entries");
                return;
            }
        }
    }

    mHashes = new int[size];
    mArray = new Object[size<<1];
}
```

## 小结

优点：

- 二分查找加快定位 key
- 缓存优化

应用场景：

- 数据结构类型为 Map 类型
- 数据量不大，最好在千级以内

如果要兼容 aip19 以下版本的话，需要使用 android.support.v4.util.ArrayMap

# 其它

- <strong>ArrayMap<K,V> 替代 HashMap<K,V></strong>
- <strong>ArraySet<K,V> 替代 HashSet<K,V></strong>
- <strong>SparseArray<V> 替代 HashMap<Integer,V></strong>
- <strong>SparseBooleanArray 替代 HashMap<Integer,Boolean></strong>
- <strong>SparseIntArray 替代 HashMap<Integer,Integer></strong>
- <strong>SparseLongArray 替代 HashMap<Integer,Long></strong>
- <strong>LongSparseArray<V> 替代 HashMap<Long,V></strong>
