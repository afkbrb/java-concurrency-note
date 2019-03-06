# 第4章 Java并发包中原子操作类原理剖析

## 目录

- [原子变量操作类](#原子变量操作类)
    - [递增和递减操作代码](#递增和递减操作代码)
    - [compareAndSet方法](#compareandset方法)
    - [AtomicLong使用示例](#atomiclong使用示例)
- [JDK8中新增的原子操作类LongAdder](#jdk8中新增的原子操作类longadder)
    - [原理](#原理)
    - [源码分析](#源码分析)
- [LongAccumulator](#longaccumulator)
- [更多](#更多)

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
上述代码中，valueOffset为AtomicLong在static语句块中进行初始化时通过Unsafe类获得的本类中value属性的内存偏移值。

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

由上可知，AtomicLong的性能瓶颈是多个线程同时去竞争一个变量的更新权导致的。而LongAdder通过将一个变量分解成多个变量，让同样多的线程去竞争多个资源解决了此问题。

### 原理

![](images/03.png)

如图，LongAdder内部维护了多个Cell，每个Cell内部有一个初始值为0的long类型变量，这样，在同等并发下，对单个变量的争夺会变少。此外，多个线程争夺同一个变量失败时，会到另一个Cell上去尝试，增加了重试成功的可能性。当LongAdder要获取当前值时，将所有Cell的值于base相加返回即可。

LongAdder维护了一个初始值为null的Cell数组和一个基值变量base。当一开始Cell数组为空且并发线程较少时，仅使用base进行累加。当并发增大时，会动态地增加Cell数组的容量。

Cell类中使用了@sun.misc.Contented注解进行了字节填充，解决了由于连续分布于数组中且被多个线程操作可能造成的**伪共享**问题(关于伪共享，可查看[《伪共享（false sharing），并发编程无声的性能杀手》](https://www.cnblogs.com/cyfonly/p/5800758.html)这篇文章)。

### 源码分析

先看LongAdder的定义

```java
public class LongAdder extends Striped64 implements Serializable
```

Striped64类中有如下三个变量：
```java

transient volatile Cell[] cells;

transient volatile long base;

transient volatile int cellsBusy;
```

cellsBusy用于实现自旋锁，状态值只有0和1，当创建Cell元素、扩容Cell数组或初始化Cell数组时，使用CAS操作该变量来保证同时只有一个变量可以进行其中之一的操作。

下面看Cell的定义：

```java
@sun.misc.Contended static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long valueOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> ak = Cell.class;
            valueOffset = UNSAFE.objectFieldOffset
                (ak.getDeclaredField("value"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

将value声明伪volatile确保了内存可见性，CAS操作保证了value值的原子性，@sun.misc.Contented注解的使用解决了伪共享问题。

下面来看LongAdder中的几个方法：

- long Sum()

```java
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
sum的结果并非一个精确值，因为计算总和时并没有对Cell数组加锁，累加过程中Cell的值可能被更改。

- void reset()

```java
public void reset() {
    Cell[] as = cells; Cell a;
    base = 0L;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                a.value = 0L;
        }
    }
}
```
reset非常简单，将base和Cell数组中非空元素的值置为0.

- long sumThenRest()

```java
public long sumThenReset() {
    Cell[] as = cells; Cell a;
    long sum = base;
    base = 0L;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null) {
                sum += a.value;
                a.value = 0L;
            }
        }
    }
    return sum;
}
```
sumThenReset同样非常简单，将某个Cell的值加到sum中后随即重置。

- void add(long x)

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    // 判断cells是否为空，如果不为空则直接进入内层判断，
    // 否则尝试通过CAS在base上进行add操作，若CAS成功则结束，否则进入内层
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        // 记录cell上的CAS操作是否失败
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            // 计算当前线程应该访问cells数组的哪个元素
            (a = as[getProbe() & m]) == null ||
            // 尝试通过CAS操作在对应cell上add
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

add方法会判断cells数组是否为空，非空则进入内层，否则尝试直接通过CAS操作在base上进行add。内层代码中，声明了一个uncontented变量来记录调用longAccumulate方法前在相应cell上是否进行了失败的CAS操作。

下面重点来看longAccumelate方法：

longAccumulate时Striped64类中定义的，其中包含了初始化cells数组，改变cells数组长度，新建cell等逻辑。

```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
    int h;
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // 初始化当前线程的probe，以便于找到线程对应的cell
        h = getProbe();
        wasUncontended = true; // 标记执行longAccumulate前对相应cell的CAS操作是否失败，失败为false
    }
    boolean collide = false; // 是否冲突，如果当前线程尝试访问的cell元素与其他线程冲突，则为true
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        // 当前cells不为空且元素个数大于0则进入内层，否则尝试初始化
        if ((as = cells) != null && (n = as.length) > 0) {
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {       // 尝试添加新的cell
                    Cell r = new Cell(x);
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {               // Recheck under lock
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)  // 如果已经进行了失败的CAS操作
                wasUncontended = true; // 则不调用下面的a.cas()函数（反正肯定是失败的）,而是重新计算probe值来尝试
            else if (a.cas(v = a.value, ((fn == null) ? v + x :fn.applyAsLong(v, x))))
                break;
            else if (n >= NCPU || cells != as)
                collide = false; // 如果当前cells长度大于CPU个数则不进行扩容，因为每个cell都使用一个CPU处理时性能才是最高的
                                 // 如果当前cells已经过时（其他线程对cells执行了扩容操作，改变了cells指向），也不会扩容
            else if (!collide)
                collide = true;  // 执行到此处说明a.cas()执行失败，即有冲突，将collide置为true，
                                 // 跳过扩容阶段，重新获取probe，到cells不同位置尝试cas，再次失败则扩容
            // 扩容
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    if (cells == as) {      
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue; // 扩容后再次尝试（扩容后cells长度改变，根据(n - 1) & h计算当前线程在cells中对应元素下标会变化，减少再次冲突的可能性）
            }
            h = advanceProbe(h); // 重新计算线程probe，减小下次访问cells元素时的冲突机会
        }
        // 初始化cells数组
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                           
                if (cells == as) {
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        // 尝试通过base的CAS操作进行add，成功则结束当前函数，否则再次循环
        else if (casBase(v = base, ((fn == null) ? v + x :fn.applyAsLong(v, x))))
            break; 
    }
}

```
代码比较复杂，细节的解释都写在注释中了。大体逻辑就是判断cells是否为空或者长度为0：如果空或者长度为0则尝试进行cells数组初始化，初始化失败的话则尝试通过CAS操作在base上进行add，仍然失败则重走一次流程；如果cells不为空且长度大于0，则获取当前线程对应于cells中的元素，如果该元素为null则尝试创建，否则尝试通过CAS操作在上面进行add，仍失败则扩容。

## LongAccumulator

LongAdder是LongAccumulator的特例，两者都继承自Striped64。

看如下代码：

```java
public LongAccumulator(LongBinaryOperator accumulatorFunction,
                        long identity) {
    this.function = accumulatorFunction;
    base = this.identity = identity;
}

public interface LongBinaryOperator {
    long applyAsLong(long left, long right);
}
```
LongAccumulator构造器允许传入一个双目运算符接口用于自定义加法规则，还允许传入一个初始值。

自定义的加法函数是如何被应用的呢？以上提到的longAccumulate()方法中有如下代码：

```java
a.cas(v = a.value, ((fn == null) ? v + x :fn.applyAsLong(v, x)))
```

LongAdder的add()方法中调用longAccumulate()方法时传入的是null，而LongAccumulator的accumulate()方法传入的是this.function，即自定义的加法函数。

## 更多

相关笔记：[《Java并发编程之美》阅读笔记](/README.md)
