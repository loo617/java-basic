# HashMap
## Base 1.7
### 初始化
HashMap map = new HashMap();
```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

static final float DEFAULT_LOAD_FACTOR = 0.75f;

public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
        //容量值过大过小校验....
        this.loadFactor = loadFactor;
        threshold = initialCapacity;
        init();
    }
```
### put
```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    //.....
}

//膨胀数组
private void inflateTable(int toSize) {
    //折算一个适合的数组长度，使数组的长度为大于等于输入长度并且是2的幂数的整数
    int capacity = roundUpToPowerOf2(toSize);
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
    //.....
}
```
再回到put函数中
```java
public V put(K key, V value) {
    //....
    //可以存放空值
    if (key == null)
        return putForNullKey(value);
    //1.
    int hash = hash(key);
    //2.
    int i = indexFor(hash, table.length);
    //寻找是否有相同的key,有则覆盖
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    //3.
    addEntry(hash, key, value, i);
    return null;
}

//1.计算hash值
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }
    //取对象的hash值
    h ^= k.hashCode();

    //对hashCode进行扰动计算，防止高位不同地位相同的hashCode导致哈希冲突。简单的说就是将高位特征和地位特征结合起来，降低哈希冲突的概率
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

//2.得到数组中的索引
static int indexFor(int h, int length) {
    //位运算的效率要高于取模运算,这里得到的length二进制每个位数都为1这样能降低哈希冲突的概率
    return h & (length-1);
}

//3.添加新key
void addEntry(int hash, K key, V value, int bucketIndex) {
    //容量超过阈值并且该数组索引不为空
    if ((size >= threshold) && (null != table[bucketIndex])) {
        //扩容2倍
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }
    //4.
    createEntry(hash, key, value, bucketIndex);
}
//4.创建新Entry链表节点，添加到数组索引中
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```
put流程:
1.判断table是否为空，为空则inflateTable()膨胀数组
2.判断key是否为null,为null则调用putForNullKey方法
3.否则计算hash值并取模得到数组索引位置
4.遍历链表准备插入,遍历过程中若发现key已经存在直接覆盖value即可
5.如果没遍历到则直接采用头插法插入节点

扩容流程:
1.初始化新table容量为旧table的两倍
2.依次遍历旧table中的节点放入新table中
3.最后更新新阈值

在多线程的情况下，头部插入会导致链表死循环，从而引起cpu100%的情况
## Base 1.8
### 初始化
HashMap map = new HashMap()
```java
//所有参数都是默认值
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

//位运算效率高
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

static final float DEFAULT_LOAD_FACTOR = 0.75f;

//树调整阈值
static final int TREEIFY_THRESHOLD = 8;

//扩容与树调整前置判读阈值
static final int MIN_TREEIFY_CAPACITY = 64;
```
### put
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //初始化时当前table为空，则先调整
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //.....
}

final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //...
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //..
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //...
    return newTab;
}
```
再回到putVal函数中
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    //...
    //判断当前数组索引是否为空，为空则新增节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //
    else {
        Node<K,V> e; K k;
        //是链表且与链表中第一个节点的key值相等，赋值给e
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //如果当前节点是二叉树，则新增节点
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //是链表且与链表中第一个节点的key值不相等，则遍历
        else {
            for (int binCount = 0; ; ++binCount) {
                //没有遍历到与key值相等的节点
                if ((e = p.next) == null) {
                    //尾插
                    p.next = newNode(hash, key, value, null);
                    //如果链表中节点数大于8则进行树化链表
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //遍历到赋值给e，直接退出
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //替换e的value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //数组使用量超过阈值则扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
put流程:
1.判断table是否为空，为空则resize()扩容
2.根据key的hash值取模得到数组索引位置
3.判断该位置是否为空，为空则直接插入
4.否则判断table[i]的首个元素是否和key一样，如果相同则覆盖value
5.否则继续判断当前数组索引位置是否为treeNode，是的话则直接红黑树插入
6.否则遍历列表准备插入
7.如果列表长度大于8,则把链表转为红黑树，否则链表插入,遍历过程中若发现key已经存在直接覆盖value即可
8.执行插入后会判断，实际存在的键值对数量是否大于最大容量threshold,如果超过则进行扩容

扩容流程:
1.初始化新table容量为旧table的两倍
2.依次遍历旧table中的节点放入新table中（1.8在这里做了优化，进一步提高了效率扩容后的位置=原位置OR=原位置+旧容量）
3.最后更新新阈值


![Alt text](/pic/20191122110823.png)


