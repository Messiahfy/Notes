## 1. SparseArray
构造函数：
```
// 分别用一个数组来放key和value
private int[] mKeys;
private Object[] mValues;

// 默认容量为10
public SparseArray() {
    this(10);
}

// mKeys 和mValues 两个数组初始化为
public SparseArray(int initialCapacity) {
    if (initialCapacity == 0) {
        mKeys = EmptyArray.INT;
        mValues = EmptyArray.OBJECT;
    } else {
        mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
        mKeys = new int[mValues.length];
    }
    mSize = 0;
}
```

put 方法：
```
public void put(int key, E value) {
    // 二分查找 mKeys 中值为 key 的位置
    // 如果返回负数，则为key应该插入的index的取反值
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        // 找到就直接设为新的值
        mValues[i] = value;
    } else {
        // 没有找到，先取反恢复为key应该插入的位置
        i = ~i;
        // 如果该位置的值已被删除，则将它更新为当前设置的key和value
        // remove方法会把值设置为DELETED
        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }
        // mGarbage也是在remove中赋值为true，表示存在需要回收的数据。
        // 并且当前数据量已经达到最大容量，则需要调用gc()回收数据
        if (mGarbage && mSize >= mKeys.length) {
            gc();

            // Search again because indices may have changed.
            // 调用gc由于删除了DELETED数组改变,需重新获取插入位置i
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }

        // 插入或扩容，如果未达到数组长度，就将数组中 i 位置开始的所有数据后移一位，然后把key放到 i 位置
        // 如果达到数组长度，则扩容：urrentSize <= 4 ? 8 : currentSize * 2
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}
```

gc() 函数，会把DELETED的数据去掉，把正常的数据前移填到原本DELETE的位置
```
private void gc() {
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
}
```
1. key只能是int，避免对key的装箱。还有一些SparseIntArray、SparseBooleanArray之类的变体，对应value是基本类型的情况，进一步减少装箱
2. key和value的数组一一对应，key按int大小顺序，二分查找，时间换空间，适用于数据量很小的场景。
3. 删除元素时，只是将其设置为DELETE，减少数组的数据移动次数，只在达到容量时才统一回收。
4. 不需要HashMap那样额外创建Node对象

## 2. ArrayMap
扩容为n=old < 4 ? 4 : (old < 8 ? 8: old*1.5)

```
// 全局的缓存，都是static
private static final int CACHE_SIZE = 10; // 缓存的数组最多10个
static Object[] mBaseCache; //用于缓存大小为4的数组
static int mBaseCacheSize; // 当前缓存小大为4的数组的数量
static Object[] mTwiceBaseCache; //用于缓存大小为8的数组
static int mTwiceBaseCacheSize; // 当前缓存小大为8的数组的数量


// 保存每个key的hashCode的数组
int[] mHashes;
// 存储key、value的数组
Object[] mArray;
```
* mHashes中从小到大排序放每个key对应的hash值，mArray中存放key和value，并且存储方式是key,value,key,value......，长度是mHashes的两倍
* 为了减少频繁地创建和回收，特意设计了两个缓存池，分别缓存大小为4和8的ArrayMap对象，可能是扩容到4或者8，也可能是自己创建ArrayMap时设置容量4或者8，来利用缓存。
* 扩容方式：size < 4，扩容到4；size >= 4 且 < 8，扩容到8；size >= 8，扩容到原来的1.5倍

构造函数：
```
public ArrayMap(int capacity, boolean identityHashCode) {
    mIdentityHashCode = identityHashCode;

    if (capacity < 0) {
        // ArrayMap中把容量为负数考虑成不可变的，禁止添加元素
        mHashes = EMPTY_IMMUTABLE_INTS;
        mArray = EmptyArray.OBJECT;
    } else if (capacity == 0) {
        // 容量为0，使用空数组
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
    } else {
        // 否则申请创建数组
        allocArrays(capacity);
    }
    mSize = 0;
}
```

申请数组，复用或新建：
```
private void allocArrays(final int size) {
    // 不可变的，禁止申请数组
    if (mHashes == EMPTY_IMMUTABLE_INTS) {
        throw new UnsupportedOperationException("ArrayMap is immutable");
    }
    // 如果申请容量为8，考虑使用缓存的数组
    if (size == (BASE_SIZE*2)) {
        synchronized (sTwiceBaseCacheLock) {
            if (mTwiceBaseCache != null) {
                final Object[] array = mTwiceBaseCache;
                mArray = array;
                try {
                    mTwiceBaseCache = (Object[]) array[0];
                    mHashes = (int[]) array[1];
                    if (mHashes != null) {
                        array[0] = array[1] = null;
                        mTwiceBaseCacheSize--;
                        return;
                    }
                } catch (ClassCastException e) {
                }
                // 如果没有可用的数组，可能是并发操作导致的，就还是会走最后的正常创建数组流程
                mTwiceBaseCache = null;
                mTwiceBaseCacheSize = 0;
            }
        }
    // 如果申请容量为4，考虑使用缓存的数组
    } else if (size == BASE_SIZE) {
        synchronized (sBaseCacheLock) {
            if (mBaseCache != null) {
                final Object[] array = mBaseCache;
                mArray = array;
                try {
                    mBaseCache = (Object[]) array[0];
                    mHashes = (int[]) array[1];
                    if (mHashes != null) {
                        array[0] = array[1] = null;
                        mBaseCacheSize--;
                        return;
                    }
                } catch (ClassCastException e) {
                }
                mBaseCache = null;
                mBaseCacheSize = 0;
            }
        }
    }
    // 没有缓存或者不是申请4或者8的容量的时候，就直接创建新数组
    mHashes = new int[size];
    mArray = new Object[size<<1];
}
```
这里的复用，需要了解一下ArrayMap缓存的方式，在稍后再了解。

这里继续看 put() 方法：
```
public V put(K key, V value) {
    final int osize = mSize;
    final int hash;
    int index;

    // 如果key为null，hash就直接用0
    if (key == null) {
        hash = 0;
        // 然后查找mHashes中，hash为0的index（不一定找到，找不到就是负数）
        index = indexOfNull();
    } else {
        // 如果key不是null，就正常的使用hashCode作为hash值
        hash = mIdentityHashCode ? System.identityHashCode(key) : key.hashCode();
        // 通过正常的hash值去 mHashes 数组中找到对应的index，乘2就是 mArray 中放的key的位置
        index = indexOf(key, hash);
    }
    if (index >= 0) {
        // 找到了 key 对应的位置，也就是之前放过了相同的key，那么就在该位置覆盖value
        index = (index<<1) + 1;
        final V old = (V)mArray[index];
        mArray[index] = value;
        return old;
    }

    // index 为负数，就是没有找到该 key，说明之前没有放过 key 相同的数据，所以就可以取反得到key应该放入 mHashes 的那个位置index
    index = ~index;

    // 如果达到容量上限
    if (osize >= mHashes.length) {
        // 如果原本size < 4，扩容到4
        // 如果原本size >= 4 且 < 8，扩容到8
        // 如果原本size >= 8，扩容到原来的1.5倍
        final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);
        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;

        // 用新的容量去申请数组，mHashes 和 mArray 都会指向新的数组
        allocArrays(n);
        if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
            throw new ConcurrentModificationException();
        }

        // 将旧的数据复制到新的数组中
        if (mHashes.length > 0) {
            System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
            System.arraycopy(oarray, 0, mArray, 0, oarray.length);
        }

        // 回收旧的数组
        freeArrays(ohashes, oarray, osize);
    }

    // 把index开始的数据全部后移一个位置
    if (index < osize) {
        System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
        System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
    }
    if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
        if (osize != mSize || index >= mHashes.length) {
            throw new ConcurrentModificationException();
        }
    }
    // 把key对应的hash，以及key和value放入数组对应的位置
    mHashes[index] = hash;
    mArray[index<<1] = key;
    mArray[(index<<1)+1] = value;
    mSize++;
    return null;
}
```

查找 mHashes 中key为null的index：
```
int indexOfNull() {
    final int N = mSize;

    // 如果数据为空，就不需要找了，直接返回0的取反值
    if (N == 0) {
        return ~0;
    }

    // key为null，hash使用0，所以在 mHashes 中二分查找0的位置
    // 如果找到了，就返回该位置，一个非负树
    // 如果没有找到，就返回它应该放入的位置index的取反值
    int index = binarySearchHashes(mHashes, N, 0);

    // 小于0表示没有找到，直接返回该负数
    if (index < 0) {
        return index;
    }

    // 如果 mArray 中，index * 2 位置的key正好是null，那么就找到了该位置
    if (null == mArray[index<<1]) {
        return index;
    }
    // hash同样为0，但是对应的key并不是null，也就是哈希冲突，就要从index+1往右遍历查找
    int end;
    for (end = index + 1; end < N && mHashes[end] == 0; end++) {
        if (null == mArray[end << 1]) return end;
    }
    // 哈希冲突往右没有找到，就再从index-1往左遍历查找
    for (int i = index - 1; i >= 0 && mHashes[i] == 0; i--) {
        if (null == mArray[i << 1]) return i;
    }
    // 都没有找到的话，返回的就是左边第一个hash为0的位置
    // 因为二分查找只是找到值相等的即可，但可能是相等的连续几个数据其中1个，所以这里是把这连续几个哈希冲突的值对应的key都遍历了，实在找不到就返回最左边的
    return ~end;
}
```
indexOf 查找key非null的index，流程也类似，这里不列出源码。


ArrayMap会在 clear()、remove()等方法中缓存数组：
```
public void clear() {
    if (mSize > 0) {
        final int[] ohashes = mHashes;
        final Object[] oarray = mArray;
        final int osize = mSize;
        mHashes = EmptyArray.INT;
        mArray = EmptyArray.OBJECT;
        mSize = 0;
        freeArrays(ohashes, oarray, osize);
    }
    if (CONCURRENT_MODIFICATION_EXCEPTIONS && mSize > 0) {
        throw new ConcurrentModificationException();
    }
}
```
回收 mHashes 和 mArray：
```
private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
    // 缓存大小为8的数组
    if (hashes.length == (BASE_SIZE*2)) {
        synchronized (sTwiceBaseCacheLock) {
            if (mTwiceBaseCacheSize < CACHE_SIZE) {
                array[0] = mTwiceBaseCache;
                array[1] = hashes;
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                mTwiceBaseCache = array;
                mTwiceBaseCacheSize++;
            }
        }
        // 缓存大小为4的数组
    } else if (hashes.length == BASE_SIZE) {
        synchronized (sBaseCacheLock) {
            if (mBaseCacheSize < CACHE_SIZE) {
                array[0] = mBaseCache;
                array[1] = hashes;
                for (int i=(size<<1)-1; i>=2; i--) {
                    array[i] = null;
                }
                mBaseCache = array;
                mBaseCacheSize++;
            }
        }
    }
}
```
缓存的方式是：让 mArray 数组的位置0引用原本的mBaseCache，位置1引用 mHashes，然后让mBaseCache引用这个 mArray，从而形成链表结构。mBaseCache引用一个 array，这个array的1引用自己的hash数组，0引用另一个array，缓存的多个数组就这样关联起来。