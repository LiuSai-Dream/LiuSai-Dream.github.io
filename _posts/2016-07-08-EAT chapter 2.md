---
layout: post
title: Efficient Andriod Threading - Chapter 2
category: Android
---

## 2. Multithreading in java
包括的内容：

- java中的并发并发编程模型
- 多线程中保持数据一致性

### Thread Basis
---

#### Execution

在`run()`方法中调用的任何局部变量（直接或者间接调用），都会存储在该线程的本地内存栈上。
（注意这样会产生内存泄漏）

在操作系统层面，线程具有指令和栈指针。指令指针指向下一个要执行的命令，栈指针指向私有的内存区域，保存着线程局部变量。


#### Single-Threaded Application

#### Multithreaded Application
如果执行的线程数量多于处理器的数量，那么就不会是真正的并行。

##### Increased resource consumption
线程带来了内存和处理器的使用开销。每个线程会分配私有的内存用来存储本地变量和参数。 处理器会在线程上下文切换的时候有开销。

##### Increased complexity

##### Data inconsistency
不仅仅是因为语句执行的顺序不保证，单条语句的也是多条指令构成的。可以通过**atomic regions**来保证指令的执行不会被其他线程突然插入。但是最根本的同步是通过**synchronized**来实现的。

### Thread Safety
多个进程同时会访问的区域被称为*critical section*，必须原子执行。

Android中的锁机制，
内部锁：

- synchronized 
外部锁：

- `java.util.concurrent.locks.ReentrantLock`
- `java.util.concurrent.locks.ReentrantReadWriteLock`


#### Intrinsic Lock and Java Monitor
*synchronized*作用在每个java对象的内部锁。*lock*作为一个*monitor*。有三种状态：

- *Blocked* 线程暂停，等待其他的线程释放锁
- *Executing* 唯一拥有*monitor*的线程在关键区域内执行
- *Waitting* 在执行完关键区域之前，线程被迫放弃监视器

#### Synchronized Access to Shared Resources

##### Using the intrinsic lock

- Method level that operates on the instrinsic lock of the enclosing object instance
```
synchronized void changeState(){ ... }
```
- Block-level that operators on the instrinsic lock of the enclosing object instance
```
void changeState() {
    synchronized(this) { ... }
}
```
- Block-level with other object intrinsic lock of the enclosing object instance
```
private final Object mLock = new Object()
void changeState() {
    synchronized(mLock) { ... }
```
- Method-level that operators on the intrinsic lock of the enclosing class instance
```
synchronized static void changeState() {
    ...
}
```
> 注意在static方法上进行同步，实际上获取的是*class object*的内部锁而非*instance object*的内部锁。

- Block-level that operators on the instrinsic lock of the enclosing class instance
```
static void changeState(){
    synchronized(this.getClass()) { ... } 
}
```

##### Using explicit locking mechanisms
更高级的同步策略是`ReentrantLock`和`ReentrantReadWeiterLock`。
其中`synchronized`和`ReentrantLock`有相同的语义，而`ReentrantReadWriterLock`则允许并发读。

使用`ReentrantReadWriterLock`的场景是读多写少的情况。

##### Concurrent Execution Design
准则：

- 重用线程而不是一直新建
- 不开启超过需要的新线程
