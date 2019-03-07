# 第5章 Java并发包中并发List源码剖析

## 介绍

JUC包中的并发List只有CopyOnWriteArrayList。CopyOnWriteArrayList是一个线程安全的ArrayList，使用了写时设置策略，对其进行的修改操作都是在底层的一个复制的数组上进行的。

## 源码解析

### 初始化

