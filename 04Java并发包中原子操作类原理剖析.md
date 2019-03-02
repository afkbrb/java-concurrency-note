# 第4章 Java并发包中原子操作类原理剖析

## 原子变量操作类

JUC包中有AtomicInteger、AtomicLong和AtomicBoolean等原子性操作类，它们原理类似，下面以AtomicLong为例进行讲解。

### 递增和递减操作代码

```java
public final long getAndIncrement() {
    return unsafe.getAndAddLong(this, valueOffset, 1L);
}

public final long getAndDecrement() {
    return unsafe.getAndAddLong(this, valueOffset, -1L);
}

public final long incrementAndGet() {
    return unsafe.getAndAddLong(this, valueOffset, 1L) + 1L;
}

public final long decrementAndGet() {
    return unsafe.getAndAddLong(this, valueOffset, -1L) - 1L;
}
```
上述代码中，valueOffset为AtomicLong在static语句块中进行初始化时通过Unsafe类获得的本类中value属性的内存偏移。

可以看到，上述四个方法都是基于Unsafe类中的getAndAddLong方法实现的。

getAndAddLong源码如下

```java
public final long getAndAddLong(Object var1, long var2, long var4) {
    long var6;
    //CAS操作设置var1对象偏移为var2处的值增加var4
    do {
        var6 = this.getLongVolatile(var1, var2);
    } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));

    return var6;
}
```

### compareAndSet方法

```java
public final boolean compareAndSet(long expect, long update) {
    return unsafe.compareAndSwapLong(this, valueOffset, expect, update);
}
```

可见，内部还是调用了Unsafe类中的CAS方法。

### AtomicLong使用示例

```java
public class AtomicLongDemo {
    private static AtomicLong al = new AtomicLong(0);

    public static long addNext() {
        return al.getAndIncrement();
    }

    public static void main(String[] args) {
        for (int i = 0; i < 100; i++) {
            new Thread() {
                @Override
                public void run() {
                    AtomicLongDemo.addNext();
                }
            }.start();
        }

        // 等待线程运行完
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("final result is " + AtomicLongDemo.addNext());
    }
}
```
AtomicLong使用CAS非阻塞算法，性能比使用synchronized等的阻塞算法实现同步好很多。但在高并发下，大量线程会同时去竞争更新同一个原子变量，由于同时只有一个线程的CAS会成功，会造成大量的自旋尝试，十分浪费CPU资源。因此，JDK8中新增了原子操作类LongAdder。

## JDK8中新增的原子操作类LongAdder

由上可知，AtomicLong的性能瓶颈是多个线程同时去竞争一个变量的更新权导致的，而LongAdder通过将一个变量分解成多个变量，让同样多的线程去竞争多个资源解决了此问题。



