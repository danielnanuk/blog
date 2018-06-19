---
title: HashMap介绍
date: 2018-06-19 09:59:33
tags: 
    - Java
    - Data Structure
---
# HashMap介绍
  HashMap是一个散列表, 它存储的内容是键值对(key-value).
  HashMap继承于AbstractMap, 实现Map、Cloneable、Serializable接口:
  ![HashMap](/images/HashMap.png  "HashMap")
# 构造器 
  HashMap提供了4个构造器:
```java
    // 无参构造器, 加载因子默认为0.75, 初始容量默认为16
    public HashMap() {}
    
    // 指定初始容量, 加载因子默认为0.75
    public HashMap(int initialCapacity) {}
    
    // 指定初始容量, 加载因子
    public HashMap(int initialCapacity, float loadFactor) {}
    
    // 使用指定Map创建, 加载因子默认为0.75
    public HashMap(Map<? extends K, ? extends V> m){}
```
构造器中有两个重要的参数: ``initialCapacity(初始容量)``和``loadFactor(加载因子)``, 其中初始容量表示hash表中桶的数量, 
加载因子表示在其容量进行扩容之前, 最大能够达到的装填程度. 加载因子越大, 空间利用率越大, 但查找效率更低.
## 存储结构
存储结构如下:
![HashMap](/images/HashMapDataStructure.png  "存储结构")

使用数组存储Entry:
![Entry](/images/Entry.png  "Entry")
其中Node为链表实现, TreeNode为红黑树实现(since 1.8).
## 静态成员
```java
// 默认初始容量, 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 最大容量 
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 哈希桶树形化阈值, 当桶的长度大于这个值时尝试进行树形化
static final int TREEIFY_THRESHOLD = 8;
// 哈希桶链表化阈值, 当桶的长度小于这个值进行链表化
static final int UNTREEIFY_THRESHOLD = 6;
// 树形化最小容量, 当树形化时, size小于该值, 进行resize而不是树形化
static final int MIN_TREEIFY_CAPACITY = 64;
```
## 实例成员
```java
// 存储元素
transient Node<K,V>[] table; 
// 缓存的entrySet, 源码见内部类EntrySet
transient Set<Map.Entry<K,V>> entrySet;
// 当前存储的元素数量 
transient int size; 
// 被结构化修改的次数, 用于在被遍历是进行fast-fail
transient int modCount; 
// 下次扩容时的阈值 
int threshold; 
// 加载因子
final float loadFactor; 
```
# 源码解析
## putVal
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 当前哈希表为空, 调用resize进行初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 对应哈希桶为空, 直接创建新node放入该桶中
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
           
            Node<K,V> e; K k;
            // 对应的key存在
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                // 如果节点是TreeNode, 则调用TreeNode的putTreeVal
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 链表, 不断遍历链表 
                for (int binCount = 0; ; ++binCount) {
                    // 找到最后一个节点
                    if ((e = p.next) == null) {
                        // 创建新节点并添加到链尾
                        p.next = newNode(hash, key, value, null);
                        // 若该桶的长度大于树形化阈值, 树形化
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // key已经存在
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 如果key存在
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 若元素数量超过阈值, 进行扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

![HashMap put](/images/put.png  "put")

## treeifyBin
```java
//将桶内所有的 链表节点 替换成 红黑树节点 
final void treeifyBin(Node<K,V>[] tab, int hash) {
  int n, index; Node<K,V> e; 
   //如果当前哈希表为空, 或者哈希表中元素的个数小于 进行树形化的阈值(默认为 64), 就去新建/扩容 
  if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY) 
       resize(); 
   else if ((e = tab[index = (n - 1) & hash]) != null) {
       //如果哈希表中的元素个数超过了 树形化阈值, 进行树形化 
       // e 是哈希表中指定位置桶里的链表节点, 从第一个开始 
       TreeNode<K,V> hd = null, tl = null; //红黑树的头、尾节点 
        do {
            //新建一个树形节点, 内容和当前链表节点 e 一致 
            TreeNode<K,V> p = replacementTreeNode(e, null); 
            if (tl == null) //确定树头节点 
                hd = p; 
           else {
               p.prev = tl; 
                tl.next = p; 
            } 
            tl = p; 
        } while ((e = e.next) != null); 
        // 让桶的第一个元素指向新建的红黑树头结点, 以后这个桶里的元素就是红黑树而不是链表了 
        if ((tab[index] = hd) != null) 
            hd.treeify(tab); 
    } 
 } 
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next); 
 }
```

## resize

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            // 根据threshold进行初始化, threshold在构造器中通过设置initialCapacity来指定
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 默认初始化
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            // 根据新容量计算threshold
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        // 创建新的table
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        // 遍历原table, 将节点放置到新table中
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 新增的那个bit是0,  位置不变
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 新增的那个bit为1, 位置等于原位置 + oldCap
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 原来的单链表变成两个链表, 放到新哈希表中的两个位置 j / j + oldCap
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
1.8对该方法进行了优化, 扩容时计算元素的位置进行了简化, 只需要判断新增出来的那个bit是0还是1, 不用再计算 (size - 1) & hash来确定位置.
更多关于1.8的优化, [https://tech.meituan.com/java-hashmap.html](https://tech.meituan.com/java-hashmap.html) 

# 小结
 + 什么时候HashMap会进行扩容?
    + 当HashMap存储的数据量大于threshold时会进行扩容
    + 当某个哈希桶中的元素数量大于TREEIFY_THRESHOLD(8)时, 且总元素数量小于MIN_TREEIFY_CAPACITY(64)
 + 加载因子大于1会怎么样?
    + 加载因子大于loadFactor会导致threshold过大, 从而每个哈希桶装的元素过多, 查询和添加效率低下
 + 什么时候哈希桶使用链表, 什么时候使用红黑树?
    + 当某个哈希桶中的元素数量大于TREEIFY_THRESHOLD(8), 且总元素数量大于MIN_TREEIFY_CAPACITY(64), 此时会使用红黑树来替换掉对应的Node,
     可以提高查询效率
 + 如何找到对应的哈希桶
    + 使用位运算来获取对应的index, index = (n - 1) & hash, 其中n等于哈希桶的数量
 + 为什么threshold一定要是2的n次方
    + 为了性能考虑, 当容量是2的n次方时, (n-1) & hash  == hash % n, 避免使用低效的取模操作
    + 扩容时, 对链表进行rehash的过程, 可以简化为检测当前key的hashcode新增的高位是否为0或1, 当为0时, 位置保持不变, 当为1时位置为原位置加上oldCap
 + TreeNode.putTreeVal的黑科技
    + 如果key值的hashCode相同时, 会先判断key是否实现Comparable, 如果实现了就根据Comparable来进行比较, 否则使用System.identityHashCode来比较(可见方法tieBreakOrder)
 
