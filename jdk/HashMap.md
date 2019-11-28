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


# ConcurrentHashMap
## Base 1.8
### 初始化

```java
//采用延迟法初始化，在put方法里
public ConcurrentHashMap() {
}
```

### put
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            //1.判断table为空，则先初始化
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //判断当前数组索引位置为空,则使用cas在改位置插入头节点
            if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            //2.判断数组当前索引位置正在扩容，则帮助扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //锁住数组索引当前位置
            synchronized (f) {
                //判断数组索引当前位置是否被修改过
                if (tabAt(tab, i) == f) {
                    //当前位置为链表，遍历链表若存在则替换value，否则插入尾部
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                            value, null);
                                break;
                            }
                        }
                    }
                    //当前位置为二叉树，则插入树中
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //如果链表中节点大于8则树化
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //3.数量+1
    addCount(1L, binCount);
    return null;
}

//1.判断table为空，则先初始化
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            //有其他线程在扩容或初始化，则把执行权交给其他线程
            Thread.yield(); // lost initialization race; just spin
        //原子操作把sizectl设置为-1，表示当前线程要初始化了
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}

//2,辅助扩容
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    //原table不为null
    //当前节点必须为fwd节点
    //nextTable已经初始化
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        //再校验一次nextTab,table,sc,若校验成功则表示正在扩容
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
            // sc >>> RESIZE_STAMP_SHIFT) != rs 说明扩容完毕或者有其它协助扩容者
            // sc == rs + 1 表示只剩下最后一个扩容线程了，其它都扩容完毕了
            // transferIndex <= 0 扩容结束了
            // sc == rs + MAX_RESIZERS 到达最大值
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            //当前线程参加扩容,sc+1
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                //2.1扩容
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}

//2.1扩容
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //计算步长最小是16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    //nextTab还未初始化
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            //扩容为两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            //初始化失败，设置为Integer的最大值
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        //更新nextTable
        nextTable = nextTab;
        //更新转移下标就是原来tab.length
        transferIndex = n;
    }
    //新table的length 
    int nextn = nextTab.length;
    //创建一个fwd节点
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    //首次推进为true,如果为true说明需要再次推进一个目标(i--)。反之是false，那么就将当前的目标处理完
    boolean advance = true;
    //完成状态，如果为true就结束方法
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //如果当前线程可以向后推进，这个循环是递减的，每个线程进来后可以取到自己需要转移桶的下标区间
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            //此时没有需要转移的桶了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //更新transferIndex为当前线程分配任务，处理桶结点区间为(nextBound,nextIndex)
            else if (U.compareAndSwapInt
                        (this, TRANSFERINDEX, nextIndex,
                        nextBound = (nextIndex > stride ?
                                    nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // i < 0 ,表示数据迁移已经完成
        // i >= n 和 i + n >= nextn 表示最后一个线程也执行完成了,扩容完成了
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                //删除成员变量
                nextTable = null;
                //更新table
                table = nextTab;
                //更新阈值0.75n
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //表示一个线程退出扩容，更新sizectl
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //说明还有其他线程存活
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        //待迁移结点为null,用cas将结点设置为fwd
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //当前节点为fwd,表示已经处理了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            synchronized (f) {
                //校验是否被修改
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    //该节点为链表
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        //遍历赋值到ln,hn
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //将链表更新到新tab
                        setTabAt(nextTab, i, ln);
                        //将链表更新到新tab
                        setTabAt(nextTab, i + n, hn);
                        //将原tab插入fwd节点
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    //改节点为二叉树
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}

//3.数量+1
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```


