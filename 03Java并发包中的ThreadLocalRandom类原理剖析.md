# 第3章 Java并发包中的ThreadLocalRandom类原理剖析

## 目录

- [Random类及其局限性](#random类及其局限性)
    - [示例](#示例)
    - [分析](#分析)
- [ThreadLocalRandom](#threadlocalrandom)
    - [示例](#示例-1)
    - [原理](#原理)
    - [源码分析](#源码分析)
- [更多](#更多)

## Random类及其局限性

一般情况下，我们都会使用java.util.Random来生成随机数（Math.random()也是使用Random实例生成随机数）。

### 示例

```java
public static void main(String[] args) {

    Random random = new Random();
    for (int i = 0; i < 10; i++) {
        System.out.println(random.nextInt(10));
    }

}
```

### 分析

下面以nextInt(int bound) 方法为例来分析Random的源码

```java
public int nextInt(int bound) {
    //边界检测
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);

    //获取下一随机数
    int r = next(31);

    //(*)此处以特定算法根据r计算出最终结果
    ...

    return r;
}

protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    //CAS操作更新seed
    do {
        oldseed = seed.get();
        //根据老的种子计算新的种子
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```

由此可见，生成新的随机数需要两步：

- 根据老的种子生成新的种子
- 由新的种子计算出新的随机数

单线程下每次调用nextInt都会根据老的种子计算出新的种子，可以保证随机性。

但多线程下，不同线程可能拿着同一个老的种子去计算新种子，**如果**next方法因此返回相同的值的话，由于(*)处的算法是固定的，这会导致不同线程生成相同的随机数，这并非我们想要的。所以next方法使用CAS操作保证每次只有一个线程可以更新老的种子，失败的线程则重新获取，这样就解决了上述问题。

但这样处理仍有一个缺陷：当多个线程同时计算随机数来计算新的种子时，多个线程会竞争同一个原子变量的更新操作，由于该操作为CAS操作，同时只有一个线程会成功，这样会造成大量的自旋重试，导致并发性能降低。而ThreadLocalRandom可以完美解决此问题。

## ThreadLocalRandom

### 示例

```java
public static void main(String[] args) {

    Random random = ThreadLocalRandom.current();
    for (int i = 0; i < 10; i++) {
        System.out.println(random.nextInt(10));
    }

}
```

### 原理

Random的缺点在于多个线程会使用同一个原子性变量，从而导致对原子变量的竞争；而ThreadLocalRandom保证每个线程都维护一个种子变量，每个线程根据自己老的种子生成新的种子，避免了竞争问题，大大提高了并发性能。

### 源码分析

```java
static final ThreadLocalRandom instance = new ThreadLocalRandom();

public static ThreadLocalRandom current() {
    //检测是否初始化过
    //PROBE为Thread类中threadLocalRandomProb偏移
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}

static final void localInit() {
    int p = probeGenerator.addAndGet(PROBE_INCREMENT);
    int probe = (p == 0) ? 1 : p; // skip 0
    long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
    Thread t = Thread.currentThread();
    //SEED为Thread类中threadLocalRandomSeed内存偏移
    UNSAFE.putLong(t, SEED, seed);
    UNSAFE.putInt(t, PROBE, probe);
}
```
如果线程中第一次调用current()方法，则调用localInit()进行初始化设置当前线程中的threadLocalRandomProb和threadLocalRandomSeed变量。

下面来看int nextInt(int bound)方法

```java
public int nextInt(int bound) {
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);
    //根据当前Thread中的threadLocalRandomSeed变量生成新种子    
    int r = mix32(nextSeed());
    int m = bound - 1;
    if ((bound & m) == 0) // power of two
        r &= m;
    else { // reject over-represented candidates
        for (int u = r >>> 1;
                u + m - (r = u % bound) < 0;
                u = mix32(nextSeed()) >>> 1)
            ;
    }
    return r;
}

final long nextSeed() {
    Thread t; long r;
    //生成并存入新种子
    UNSAFE.putLong(t = Thread.currentThread(), SEED,
                    r = UNSAFE.getLong(t, SEED) + GAMMA);
    return r;
}
```
如上，首先调用nextSeed()根据当前Thread中的threadLocalRandomSeed变量生成并存入新种子，然后经过特定算法得出了nextInt的值。  

## 更多

相关笔记：[《Java并发编程之美》阅读笔记](/README.md)
