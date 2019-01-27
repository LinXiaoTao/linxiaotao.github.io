---
title: ArrayList源码分析
date: 2018-11-09 10:16:12
categories: Java
tags:
---

### 构造函数

ArrayList 的构造函数有三个：

* `ArrayList()`
* `ArrayList(int)`
* `ArrayList(Collection<? extends E>)`

其中用的最多的是无参默认构造函数，ArrayList 底层是使用数组实现的，这里初始化一个空数组

``` java
public ArrayList() {                                     
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
} 

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

如果传入指定的容量（initialCapacity），则会初始化指定大小的数组

### add 操作

调用 `add(value)` 会在列表的尾部添加一个元素

``` java
public boolean add(E e) {                                                  
    ensureCapacityInternal(size + 1);  // Increments modCount!!            
    elementData[size++] = e;                                               
    return true;                                                           
}

private void ensureCapacityInternal(int minCapacity) {                  
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 如果是初始化，则使用 DEFAULT_CAPACITY = 10
        return Math.max(DEFAULT_CAPACITY, minCapacity);                       
    }                                                                         
    return minCapacity;                                                       
}

private void ensureExplicitCapacity(int minCapacity) { 
    modCount++;                                        
                                                       
    // overflow-conscious code                         
    if (minCapacity - elementData.length > 0) // 需要扩容          
        grow(minCapacity);                             
}

// 可使用的数组最大容量，一些虚拟机会在数组中保留一个头部字段
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private void grow(int minCapacity) {                           
    // overflow-conscious code 考虑溢出情况                                 
    int oldCapacity = elementData.length;
    // 增加旧容量 * 0.5 = 旧容量 * 1.5
    int newCapacity = oldCapacity + (oldCapacity >> 1);        
    if (newCapacity - minCapacity < 0)                         
        newCapacity = minCapacity;     // 还不够大，直接用 minCapacity                       
    if (newCapacity - MAX_ARRAY_SIZE > 0)  // 大于允许的最大容量                    
        newCapacity = hugeCapacity(minCapacity);               
    // minCapacity is usually close to size, so this is a win: 
    elementData = Arrays.copyOf(elementData, newCapacity);     
}

private static int hugeCapacity(int minCapacity) { 
    if (minCapacity < 0) // overflow // 超过 Integer.MAX_VALUE，变成负值             
        throw new OutOfMemoryError();              
    return (minCapacity > MAX_ARRAY_SIZE) ?        
        Integer.MAX_VALUE :                        
        MAX_ARRAY_SIZE;                            
}                                                  
```

> 允许添加 null

### remove 操作

调用 `remove(int)` 删除指定下标的元素

``` java
public E remove(int index) {
    // 下标范围检查
    rangeCheck(index);                                                   
                                                                         
    modCount++;                                                          
    E oldValue = elementData(index);                                     
    
    // 需要移动的个数
    int numMoved = size - index - 1;                                     
    if (numMoved > 0)
        // 元素拷贝移动
        System.arraycopy(elementData, index+1, elementData, index,       
                         numMoved);                                      
    elementData[--size] = null; // clear to let GC do its work           
                                                                         
    return oldValue;                                                     
}                                                                        
```

调用 `remove(object)` 删除指定元素

``` java
public boolean remove(Object o) {                      
    if (o == null) {
        // 如果删除 null 元素
        for (int index = 0; index < size; index++)     
            if (elementData[index] == null) {          
                fastRemove(index);                     
                return true;                           
            }                                          
    } else {                                           
        for (int index = 0; index < size; index++)
            // 调用 equals 判断
            if (o.equals(elementData[index])) {        
                fastRemove(index);                     
                return true;                           
            }                                          
    }                                                  
    return false;                                      
} 
private void fastRemove(int index) {                                       
    modCount++;                                                            
    int numMoved = size - index - 1;                                       
    if (numMoved > 0)                                                      
        System.arraycopy(elementData, index+1, elementData, index,         
                         numMoved);                                        
    elementData[--size] = null; // clear to let GC do its work             
}                                                                                                                               
```

