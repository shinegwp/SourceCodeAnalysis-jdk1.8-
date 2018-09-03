# HashMap

- hash
```
散列，将一个任意的长度的某种算法转转化为任意的值。移位
```
- map
```
键值对存储
```
- 内部结构

```
1.数组
名为table的Node类型的数组  

2.链表
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;//hash值
    final K key;//key值
    V value;//value值
    Node<K,V> next;//下一个结点
    Node(int hash, K key, V value, Node<K,V> next) {//初始化
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
3.红黑树
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);//回退到Node的构造函数
        }

        /**
         * Returns root of tree containing this node.
         */
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }
    }

```



## 源码实现

> 初始化

```
无参
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

一个参数
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);//调用两个参数的构造方法
    }
两个参数
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)//初始化容量小于0，抛异常
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)//初始化容量的值大于最大值则赋值最大值
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))//loadFactory小于0或者不存在跑异常
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);//记录下次扩容的数量
    }
    static final int tableSizeFor(int cap) {//将值变为最接近的它的2的n次方的数
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

```

> 1.容量

```
默认16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
最大容量2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

加载因子0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;

转化为红黑树的链表长度为8
static final int TREEIFY_THRESHOLD = 8;

红黑树转换为链表的长度为6
static final int UNTREEIFY_THRESHOLD = 6;

红黑树的节点数超过64对hashmap扩容
static final int MIN_TREEIFY_CAPACITY = 64;

```

> 2.key值的计算

```
hash(key)方法（若key为null则为0，否则获取hashcode无符号右移16位）
return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);

数组下表的确定：
tab[i = (n - 1) & hash]) == null 与长度的n-1相与

```
> 3.put方法

```
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab;//创建一个node数组
        Node<K,V> p;//创建node结点
        int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;//获取一个新的node数组，并获取长度
        if ((p = tab[i = (n - 1) & hash]) == null)//与长度的n-1相与
            tab[i] = newNode(hash, key, value, null);//该位置上没有值则直接new一个新节点
        else {
            Node<K,V> e; K k;//新建一个结点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))//判断key是否是重复值
                e = p;//将原来的值赋给新节点
            else if (p instanceof TreeNode)//不是重复值时判断是否是树节点
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);//将将要赋值的树节点赋值给e.
            else {//否则仍然是一个链表
                for (int binCount = 0; ; ++binCount) {//循环找到最后一个元素，将新值加入后面
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;//若key是重复的则是返回旧值，将新值覆盖旧值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)//判断长度是否达到扩容值
            resize();//进行扩容
        afterNodeInsertion(evict);
        return null;
    }
```

> 4.get方法

```
    public V get(Object key) {
        Node<K,V> e;
        //值不存在则返回null，存在则返回对应值
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {//确定数组不为null且计算的hash值存在
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
                //循环找到key对应的值
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

> 5.resize()扩容方法

```
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;//若已经是最大容量则直接返回原数组，不扩容
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)//双倍扩容
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {//若扩容临界容量为0
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {//数据转移
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
                        do {//对同一链表上的数据进行遍历
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
                        if (loTail != null) {//将相与为0的hash放在原位置
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {//相与不为0的放在原位置+原始容量的位置
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
## 不足

```
不足：每次扩容都需要通过Key的hash值重新计算在数组中的位置，扩容时会导致时间复杂度极高
```
