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

```
