---
title: HashMap源码分析
date: 2018-11-07 11:39:06
categories: Java
tags:
---

### 构造方法

从构造方法来看，我们可以指定初始化容量（initialCapacity）和负载因子（loadFactor），其中 loadFactor 的默认值为 0.75，如果指定了 initialCapacity，就会计算容量阙值（threshold）：

``` java
// tableSizeFor 会计算一个大于或等于 initialCapacity 的 2 的 N 次方的值
this.threshold = tableSizeFor(initialCapacity);

static final int tableSizeFor(int cap) {                                     
    int n = cap - 1;                                                         
    n |= n >>> 1;                                                            
    n |= n >>> 2;                                                            
    n |= n >>> 4;                                                            
    n |= n >>> 8;                                                            
    n |= n >>> 16;                                                           
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1; 
}                                                                            
```

> initialCapacity 只会在构造函数中用到，用于计算 threshold

### put 操作

当调用 `put(key,value)` 的时候：

``` java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

首先会根据 key 调用 `hash()` 计算哈希值，可以看到 `hash()` 中我们不仅使用 `key.hashCode()` 还将哈希值无符号右移 16 位再做一次异或操作

> 在 HashMap 中，容量（capacity）是 2 的 N 次方，所以在**取余**的时候，可以用 `key & (capacity - 1)` 来代替，当 capacity 较小时，参与计算的位也比较少，比如，使用默认初始化容量 1<<4（即 16），那么计算下标：`key & (0x1111)` 参与计算的位只有 4 位，发生碰撞的概率也比较大

``` java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,                  
               boolean evict) {                                                 
    Node<K,V>[] tab; Node<K,V> p; int n, i;                                     
    if ((tab = table) == null || (n = tab.length) == 0) // 当 table 为空时候，需要调用 resize()                         
        n = (tab = resize()).length;                                            
    if ((p = tab[i = (n - 1) & hash]) == null)  // 如果当前对应 Node 为空，直接添加新的 Node 即可                                  
        tab[i] = newNode(hash, key, value, null);                               
    else {                                                                      
        Node<K,V> e; K k;                                                       
        if (p.hash == hash &&                                                   
            ((k = p.key) == key || (key != null && key.equals(k))))  // 如果 Head Node 为结果           
            e = p;                                                              
        else if (p instanceof TreeNode)    // 如果是红黑树                                     
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);     
        else {
            // 遍历链表
            for (int binCount = 0; ; ++binCount) {                              
                if ((e = p.next) == null) {
                    // 如果没有找到匹配的 Node，添加一个新的 Node
                    p.next = newNode(hash, key, value, null);                   
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st 
                        // 转换为红黑树
                        treeifyBin(tab, hash);                                  
                    break;                                                      
                }                                                               
                if (e.hash == hash &&                                           
                    ((k = e.key) == key || (key != null && key.equals(k)))) // 找到已存在的 Node    
                    break;                                                      
                p = e;                                                          
            }                                                                   
        }                                                                       
        if (e != null) { // existing mapping for key
            // key 对应的 Node 已存在
            V oldValue = e.value;                                               
            if (!onlyIfAbsent || oldValue == null)                              
                e.value = value;                                                
            afterNodeAccess(e);                                                 
            return oldValue;                                                    
        }                                                                       
    }
    // 添加新的 Node，modCount 改变
    ++modCount;                                                                 
    if (++size > threshold) // size 大于阙值，需要扩容                                                     
        resize();                                                               
    afterNodeInsertion(evict);                                                  
    return null;                                                                
}                                                                               
```

如果 table 为空的时候，会先调用 `resize()` 来做初始化

``` java
final Node<K,V>[] resize() {                                                    
    Node<K,V>[] oldTab = table;                                                 
    int oldCap = (oldTab == null) ? 0 : oldTab.length;                          
    int oldThr = threshold;                                                     
    int newCap, newThr = 0;                                                     
    if (oldCap > 0) {
        // 如果为扩容操作
        if (oldCap >= MAXIMUM_CAPACITY) {                                       
            threshold = Integer.MAX_VALUE;                                      
            return oldTab;                                                      
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&                   
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // capacity 左移一位，双倍
            // 调用 resize 初始化的时候，会将 capacity 设置为默认值 16
            newThr = oldThr << 1; // double threshold                           
    }                                                                           
    else if (oldThr > 0) // initial capacity was placed in threshold
        // 当使用指定 capacity 的构造方法时，会使用 tableSizeFor 初始化 threshold，2 的 N 次方
        newCap = oldThr;                                                        
    else {               // zero initial threshold signifies using defaults
        // 初始化
        newCap = DEFAULT_INITIAL_CAPACITY;                                      
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);         
    }                                                                           
    if (newThr == 0) {
        // 计算 threshold = capacity * loadFactor
        float ft = (float)newCap * loadFactor;                                  
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?   
                  (int)ft : Integer.MAX_VALUE);                                 
    }                                                                           
    threshold = newThr;                                                         
    @SuppressWarnings({"rawtypes","unchecked"})
    // 创建新的 table
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];                     
    table = newTab;                                                             
    if (oldTab != null) {
        // 拷贝旧的 table
        for (int j = 0; j < oldCap; ++j) {                                      
            Node<K,V> e;                                                        
            if ((e = oldTab[j]) != null) {                                      
                oldTab[j] = null;                                               
                if (e.next == null) // 如果只有一个 Node                                             
                    newTab[e.hash & (newCap - 1)] = e;                          
                else if (e instanceof TreeNode)  // 如果为红黑树                               
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);          
                else { // preserve order  保持顺序
                    // 旧下标的链表
                    Node<K,V> loHead = null, loTail = null;                     
                    // 新下标（oldIndex + oldCapacity）的链表
                    Node<K,V> hiHead = null, hiTail = null;                     
                    Node<K,V> next;                                             
                    do {                                                        
                        next = e.next;                                          
                        if ((e.hash & oldCap) == 0) {                           
                            if (loTail == null)                                 
                                loHead = e;                                     
                            else                                                
                                loTail.next = e;                                
                            loTail = e;                                         
                        }                                                       
                        else {                                                  
                            if (hiTail == null)                                 
                                hiHead = e;                                     
                            else                                                
                                hiTail.next = e;                                
                            hiTail = e;                                         
                        }                                                       
                    } while ((e = next) != null);                               
                    if (loTail != null) {                                       
                        loTail.next = null;                                     
                        newTab[j] = loHead;                                     
                    }                                                           
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

`resize()` 时如果需要扩容，会发生 table 拷贝，如果 Head Node 存储数据结构为链表时，会保持原来链表的顺序，同时使用两个新的链表去保存，一个链表为旧下标位置，一个链表为 旧下标+旧容量 位置

> 新下标要么为旧值，要么为旧值 + 旧容量，根据 (hash & oldCap == 0)，如果为 1，表示旧值+ 旧容量，因为容量为 2 的 N 次方，当 hash(key) 中和 capacity 的最高 1 位相对应的位为 1，则新容量时这个位会更新为 1，即添加 oldCap
>
> 举个例子：hash(key) = 17 即 0x10001，容量为 16，下标为 `hash(key) & (cap - 1) = 1` ，扩容后，新容量为 32，下标为 `hash(key) & (cap - 1) = 17`

在查找是否存在 key 对应的值时，先判断引用是否相等，否则，如果 key 不为 null，则调用 `equals()` 去比较 

```java
((k = p.key) == key || (key != null && key.equals(k)))
```

### remove 操作

根据 key 删除键值队，同样的先对 key 进行 hash ，再调用 `removeNode()`

``` java
final Node<K,V> removeNode(int hash, Object key, Object value,                 
                           boolean matchValue, boolean movable) {              
    Node<K,V>[] tab; Node<K,V> p; int n, index;                                
    if ((tab = table) != null && (n = tab.length) > 0 &&                       
        (p = tab[index = (n - 1) & hash]) != null) {                           
        Node<K,V> node = null, e; K k; V v;                                    
        if (p.hash == hash &&                                                  
            ((k = p.key) == key || (key != null && key.equals(k))))    // 如果头部节点 即为需要删除的节点         
            node = p;                                                          
        else if ((e = p.next) != null) {                                       
            if (p instanceof TreeNode)                                         
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);    // 红黑树节点          
            else {                                                             
                do {                                                           
                    if (e.hash == hash &&                                      
                        ((k = e.key) == key ||                                 
                         (key != null && key.equals(k)))) {                    
                        node = e;                                              
                        break;                                                 
                    }                                                          
                    p = e;                                                     
                } while ((e = e.next) != null);                                
            }                                                                  
        }                                                                      
        if (node != null && (!matchValue || (v = node.value) == value ||       
                             (value != null && value.equals(v)))) {
            // 存在需要删除的节点
            if (node instanceof TreeNode)                                      
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);  // 红黑树    
            else if (node == p)         // 删除节点为头部节点                                       
                tab[index] = node.next;                                        
            else                        // p 为删除节点的父节点                                       
                p.next = node.next;                                            
            ++modCount;                                                        
            --size;                                                            
            afterNodeRemoval(node);                                            
            return node;                                                       
        }                                                                      
    }                                                                          
    return null;                                                               
}                                                                              
```



