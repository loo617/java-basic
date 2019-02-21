# CAS与原子变量
synchronized通常用于同步代码块，当我们只需要对某个变量进行并发更新时使用synchronized开销过大，通常使用更细粒度的原子变量类Atomic，该类利用CAS机制更新变量实际上底层cpu调用cmpxchg命令，从指定层保证可靠不会被多线程干扰，CAS包含三个参数V、E、N，V表示要更新的变量，E表示预期值，N表示新值。仅当V与E相同时，V才更新成E，否则什么都不做。由于CAS采用乐观的态度操作，当前只有一个线程能够更新成功，其他线程失败但不会退出，而是被挂起并再次尝试。核心代码：
```java
int prev, next;
do {
    prev = get();
    next = accumulatorFunction.applyAsInt(prev, x);
} while (!compareAndSet(prev, next));
```

## Unsafe
compareAndSet(prev, next)方法调用的是sun.misc包下Unsafe类里的一个方法，比如：根据偏移量设置值, 线程park(), 底层的CAS操作等等。主要接口：
```java
 1 // 获得给定对象偏移量上的int值
 2 public native int getInt(Object o, long offset);
 3 // 设置给定对象偏移量上的int值
 4 public native void putInt(Object o, long offset, int x);
 5 // 获得字段在对象中的偏移量
 6 public native long objectFieldOffset(Field f);
 7 // 设置给定对象的int值，使用volatile语义
 8 public native void putIntVolatile(Object o, long offset, int x);
 9 // 获得给定对象对象的int值，使用volatile语义
10 public native int getIntVolatile(Object o, long offset);
11 // 和putIntVolatile()一样，但是它要求被操作字段就是volatile类型的
12 public native void putOrderedInt(Object o, long offset, int x);
```

## ABA问题
线程准备使用CAS将变量从A更新成B,在此之前线程二进入，将A更新成C然后又更新成A,此时CAS执行时发现变量仍为A所以执行成功。该问题会导致忽略了之前的操作，如优惠券场景中，会导致重复发放优惠券。

## AtomicStampedReference
通过将版本号引入，可以解决ABA问题。核心代码：
```java
 public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }
```

## 原子操作工具类
- 基本数据类型:AtomicInteger,AtomicLong,AtomicBoolean
- 数组类型:AtomicIntegerArray,AtomicLongArray,AtomicReferenceArray
- 引用类型:AtomicReference,AtomicReferenceFieldUpdater,AtomicMarkableReference
