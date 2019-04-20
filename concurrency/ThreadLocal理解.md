# ThreadLocal理解
## 是什么
一个解决多线程并发问题的工具类，当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，每个线程都可以独立地改变自己的副本，而不影响其他线程锁对应的副本。所以ThreadLocal是线程安全的。
### Thread与ThreadLocal之间的关系

![Alt text](/pic/1.png)

看了ThreadLocal源码，不知道大家有没有一个疑惑：为什么像上图那么设计？ 如果给你设计，你会怎么设计？相信大部分人会有这样的想法，我也是这样的想法：”每个ThreadLocal类创建一个Map，然后用线程的ID作为Map的key，实例对象作为Map的value，这样就能达到各个线程的值隔离的效果“
JDK最早期的ThreadLocal就是这样设计的。（不确定是否是1.3）之后ThreadLocal的设计换了一种方式，也就是目前的方式，那有什么优势了：
- 这样设计之后每个Map的Entry数量变小了：之前是Thread的数量，现在是ThreadLocal的数量，能提高性能，据说性能的提升不是一点两点(没有亲测)
- 当Thread销毁之后对应的ThreadLocalMap也就随之销毁了，能减少内存使用量
## 使用场景 
最常见的ThreadLocal使用场景为 用来解决数据库连接、Session管理等
### 数据库连接池
```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {  
    public Connection initialValue() {  
        return DriverManager.getConnection(DB_URL);  
    }  
};  
  
public static Connection getConnection() {  
    return connectionHolder.get();  
}
```
### Session管理
```java

private static final ThreadLocal threadSession = new ThreadLocal();  
  
public static Session getSession() throws InfrastructureException {  
    Session s = (Session) threadSession.get();  
    try {  
        if (s == null) {  
            s = getSessionFactory().openSession();  
            threadSession.set(s);  
        }  
    } catch (HibernateException ex) {  
        throw new InfrastructureException(ex);  
    }  
    return s;  
}
```
### ThreadLocal与使用数据库线程池的区别
用数据库线程池的目的是为了避免不必要的创建、销毁connection而存在的，其中包括活动线程数、最小线程数、等待线程数等，提高性能；用ThreadLocal是为了多个数据库操作是用的同一个连接，是为了事务；两个是不同的目的的。
## 缺点
可能会引起内存泄漏;ThreadLocalMap中key维护着一个weakReference,它在下次GC之前会被清理,如果Value仍然保持着外部的强引用,该ThreadLocal没有再进行set,get或者remove操作,时间长了就可能导致OutOfMemoryError .
