# 第10章 Java并发包中线程同步器原理剖析

## CountDownLatch原理剖析

日常开发中经常遇到一个线程需要等待一些线程都结束后才能继续向下运行的场景，在CountDownLatch出现之前通常使用join方法来实现，但join方法不够灵活，所以开发了CountDownLatch。

### 示例

```java
public static void main(String[] args) throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(2);

    ExecutorService executorService = Executors.newFixedThreadPool(2);
    // 添加任务
    executorService.execute(new Runnable() {
        @Override
        public void run() {
            try {
                // 模拟运行时间
                Thread.sleep(1000);
                System.out.println("thread one over...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                // 递减计数器
                countDownLatch.countDown();
            }
        }
    });

    // 同上
    executorService.execute(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(1000);
                System.out.println("thread two over...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                countDownLatch.countDown();
            }
        }
    });

    System.out.println("wait all child thread over!");
    // 阻塞直到被interrupt或计数器递减至0
    countDownLatch.await();
    System.out.println("all child thread over!");
    executorService.shutdown();
}
```

输出为：

    wait all child thread over!
    thread one over...
    thread two over...
    all child thread over!

CountDownLatch相对于join方法的优点大致有两点：

- 调用一个子线程的join方法后，该线程会一直阻塞直到子线程运行完毕，而CountDownLatch允许子线程运行完毕或在运行过程中递减计数器，也就是说await方法不一定要等到子线程运行结束才返回。
- 使用线程池来管理线程一般都是直接添加Runnable到线程池，这时就没有办法再调用线程的join方法了，而仍可在子线程中递减计数器，也就是说CountDownLatch相比join方法可以更灵活地控制线程的同步。

### 类图结构

![](images/14.png)

由图可知，CountDownLatch是基于AQS实现的。

由下面的代码可知，**CountDownLatch的计数器值就是AQS的state值**。

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
Sync(int count) {
    setState(count);
}
```

### 源码解析

#### void await()

当线程调用CountDownLatch的await方法后，当前线程会被阻塞，直到CountDownLatch的计数器值递减至0或者其他线程调用了当前线程的interrupt方法。

```java
public void await() throws InterruptedException {
    // 允许中断（中断时抛出异常）
    sync.acquireSharedInterruptibly(1);
}

// AQS的方法
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // state=0时tryAcquireShared方法返回1，直接返回
    // 否则执行doAcquireSharedInterruptibly方法
    if (tryAcquireShared(arg) < 0)
        // state不为0，调用该方法使await方法阻塞
        doAcquireSharedInterruptibly(arg);
}

// Sync的方法（重写了AQS中的该方法）
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

// AQS的方法
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                // 获取state值，state=0时r=1，直接返回，不再阻塞
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 若state不为0则阻塞调用await方法的线程
            // 等到其他线程执行countDown方法使计数器递减至0
            // （state变为0）或该线程被interrupt时
            // 该线程才能继续向下运行
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

#### boolean await(long timeout, TimeUnit unit)

相较于上面的await方法，调用此方法后调用线程最多被阻塞timeout时间（单位由unit指定），即使计数器没有递减至0或调用线程没有被interrupt，调用线程也会继续向下运行。

```java
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

#### void countDown()

递减计数器，当计数器的值为0（即state=0）时会唤醒所有因调用await方法而被阻塞的线程。

```java
public void countDown() {
    // 将计数器减1
    sync.releaseShared(1);
}

// AQS的方法
public final boolean releaseShared(int arg) {
    // 当state被递减至0时tryReleaseShared返回true
    // 会执行doReleaseShared方法唤醒因调用await方法阻塞的线程
    // 否则如果state不是0的话什么也不做
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

// Sync重写的AQS中的方法
protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        // 如果state已经为0，没有递减必要，直接返回
        // 否则会使state变成负数
        if (c == 0)
            return false;
        int nextc = c-1;
        // 通过CAS递减state的值
        if (compareAndSetState(c, nextc))
            // 如果state被递减至0，返回true以进行后续唤醒工作
            return nextc == 0;
    }
}

```