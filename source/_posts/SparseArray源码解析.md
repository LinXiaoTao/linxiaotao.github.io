---
title: SparseArray源码解析
date: 2019-01-08 21:58:23
categories: Android Application
tags:
---

### 关于SparseArray

`SparseArray` 是 Android SDK 提供的将 integers 映射到对象的容器。相对于 `HashMap` 更节省内存，因为它避免了 key 发生自动装箱，同时不用额外的对象来表示映射关系。

`SparseArray` 使用二分查找来查询 key，这意味着，key 是有序存储的，而且不适用于大量数据的情况。通常情况下，它的查找比 `HashMap` 慢。

为了提高性能，`SparseArray` 在删除 key 时，不会立即调整数据结构，而是将该条目标记为删除，在之后使用相同 key 时可以复用或者垃圾回收操作中进行调整。

> 垃圾回收操作会发生在需要扩容、获取size 或者检索条目值时

### 源码分析

#### 构造函数

`SparseArray` 有两个构造函数

``` java
private int[] mKeys;
private Object[] mValues;
public SparseArray() {                                                  
    this(10);                                                           
}                                                                       
                                                                                                                                            
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

如果使用不指定初始化容量，将使用默认 10，可以看到，`SparseArray` 内部使用两个数组来分别存储 key 和 value

#### put

如果 key 没有存在，则添加，否则进行替换操作

``` java
public void put(int key, E value) {                                           
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);                 
                                                                              
    if (i >= 0) { 
        // key 已经存在，替换
        mValues[i] = value;                                                   
    } else {
        // key 不存在，取反获取 key 应该插入的下标
        i = ~i;                                                               
                                                                              
        if (i < mSize && mValues[i] == DELETED) {
            // 复用被标记为 DELETED 的条目
            mKeys[i] = key;                                                   
            mValues[i] = value;                                               
            return;                                                           
        }                                                                     
                                                                              
        if (mGarbage && mSize >= mKeys.length) { 
            // 需要扩容
            gc();                                                             
                                                                              
            // Search again because indices may have changed.                 
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);            
        }                                                                     
        
        // 插入新的条目
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);               
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);         
        mSize++;                                                              
    }                                                                         
}

static int binarySearch(int[] array, int size, int value) {       
    int lo = 0;                                                   
    int hi = size - 1;                                            
                                                                  
    while (lo <= hi) {                                            
        final int mid = (lo + hi) >>> 1;                          
        final int midVal = array[mid];                            
                                                                  
        if (midVal < value) {                                     
            lo = mid + 1;                                         
        } else if (midVal > value) {                              
            hi = mid - 1;                                         
        } else {                                                  
            return mid;  // value found                           
        }                                                         
    }                                                             
    return ~lo;  // value not present                             
}                                                                 
```

`binarySearch` 就是二分查找算法的实现 ，需要注意的是，如果没有查找到，则返回 `~lo`

> 这里的 `lo` 可以认为是，虽然没有查找到指定的值，但这是最接近的，换句话说，这个是指定值应该插入的位置
>
> 比如：在 1,2,3,4,6,7 查找 5，那么 `binarySearch` 将返回 `~4`

#### get

get 操作比较简单，同样的先二分查找 key，存在则返回，否则返回默认值

``` java
public E get(int key, E valueIfKeyNotFound) {                 
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key); 
                                                              
    if (i < 0 || mValues[i] == DELETED) {                     
        return valueIfKeyNotFound;                            
    } else {                                                  
        return (E) mValues[i];                                
    }                                                         
}                                                             
```

#### remove

remove 操作实际调用 `delete` 方法

``` java
public void delete(int key) {                                   
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);   
                                                                
    if (i >= 0) {                                               
        if (mValues[i] != DELETED) {
            // 只是标记为 DELETED
            mValues[i] = DELETED;
            // 需要回收
            mGarbage = true;                                    
        }                                                       
    }                                                           
}                                                               
```

#### size

调用 `size` 之后，会先检查是否需要 `gc` 即回收操作

``` java
public int size() {   
    if (mGarbage) {   
        gc();         
    }                 
                      
    return mSize;     
}                     
```

#### gc

在调用 `size` 和涉及 index 相关的方法时，都会检查是否需要 `gc`，比如 `keyAt` 等

``` java
private void gc() {                                          
    // Log.e("SparseArray", "gc start with " + mSize);       
                                                             
    int n = mSize;                                           
    int o = 0;                                               
    int[] keys = mKeys;                                      
    Object[] values = mValues;                               
                                                             
    for (int i = 0; i < n; i++) {
        // 将没有标记为 DELETED 的条目移到新位置，清除 DELETED
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

### 总结

`SparseArray` 是用两个数组来分别存储 key 和 value，其中 key 是 `int[]` 避免了自动装箱，同时是升序的，所以查找 key 可以使用二分查找，相对于 `HashMap` 更省内存，但检索速度更慢。在数据量较少的情况下，可以使用 `SparseArray` 代替 `HashMap`

