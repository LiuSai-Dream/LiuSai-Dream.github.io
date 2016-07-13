---
layout: post
title: Efficient Andriod Threading - Chapter 3
category: Android
---

## 3. Threads on Android

应用具有UI线程，binder线程以及自己创建的后台线程。

- UI, Binder, and background 线程之间的异同点
- Linux线程解耦
- 应用优先级如何影响线程的调度
- 运行linux线程

### Android Application Threads

所有应用线程都是基于linux中原生的`pthreads`，在java中的表现形式为`Thread`。但是*platform*赋予了不同的特征把线程分为UI，binder, background线程。

#### UI Thread
UI线程是应用的主线程，用来执行部件更新UI元素。UI线程不是线程安全的。

UI线程是一个**顺序事件处理线程**（sequential event handler thread），可以执行从其他线程发过来的事件。
事件被顺序执行，如果UI线程在处理其他的事件，那么新到的事件会被入队列。

#### Binder Threads

Binder线程用来**跨进程通信**。每个进程都会维护一个线程池，该线程池**不会终止**或者**重建**。这些线程会处理**其他进程**发过来的请求。包括system service，intents, content providers， services。 如果需要，会创建新的binder thread处理请求。
大多数情况下，应用不需要考虑binder线程，因为平台通常会把请求首先转换为使用UI线程。例外是当应用提供对外的service的时候。

#### Background Threads

应用显式创建的都是后台线程。后台线程 are descendants of UI thread，它们继承了UI线程的特性。例如priority。

在应用中，UI线程和workder线程有很大的不同，但是在linux中它们**都是原生的线程**，没有区别。对UI线程的约束是应用框架中的*WindowManager*添加的而非linux对其约束的。

### The Linux Process and Threads

每个运行的应用都有底层的linux 进程，从*Zygote*进程fork得到。具有如下的性质：

- *User ID(UID)*：一个进程有唯一的User id。每个应用在系统中代表了一个用户。当应用被安装时，就会被赋予一个user ID。 

- *Process identifier(PID)*：进程唯一标识符

- *Parent process identifier(PPID)*：在系统启动后，每个进程都是从另外一个进程中创建得来的，形成一个树状结构。因此每个应用进程都有一个父进程。在Android中，该父进程是*Zygote*

- *Stack*：局部函数和变量指针
- *Heap*：

#### Finding Application Process Information
运行的应用进程信息通过ADB shell中的`ps`命令来得到。
其选项：

- -t 显示进程中的线程信息
- -x 显示运行在用户代码上的时间（utime）和系统代码上的时间（stime）
- -p 显示优先级
- -P 显示调度策略，通常预示着当前的应用是在前台还是后台运行
- -c 显示哪个CPU正在执行该进程
- name|pid 过滤应用的名称或者进程ID

也可以使用`grep`命令进行过滤。

线程遵循POSIX标准。线程属于创建它们的进程，每个线程的双亲是进程。在同一进程内的线程因为共享地址空间，通信非常容易。而进程之间的线程则需要远程过程调用。
应用中的大部分线程是Dalvik的内部线程，例如垃圾收集一类的。

当应用启动的时候，UI线程最先开始，其他的线程都是从UI线程催生的。可以看到其他线程*PPID*是UI线程的*PID*。

### Scheduling

linux把线程而非进程作为调度执行单元。
Android中，应用的所有线程是通过**linux kernel**而**非Dalvik虚拟机**进行调度。 实际中，这就意味着一个应用中的线程与所有应用中的线程进行竞争。

linux kernel的调度是*completely fair scheduler*。结合**优先级**和**运行时间**进行调度。

平台主要是有两种方式来影响线程的调度：

- *priority*。线程从其双亲获得优先级，直到应用对其进行修改。Android中使用了linux中的优先级，然而java的优先级与linux的优先级有对应的关系。
两种方式改变优先级：

    - `java.lang.Thread` 的 `setPriority(int priority)`
    - `android.os.Process` 的 `Process.setThreadPriority(int priority)`; `Process.setThreadPriority(int threadId, int priority)`
    
- *Control group*。Android既依赖linux的**CFS**进行线程调度，同时赋予所有线程**线程控制组**。
线程控制组是**linux容器**，用来管理在一个容器中的所有线程的执行时间。应用中的所有线程属于其中一个线程控制组。
Android定义多个control groups，最重要的是*Foreground group*和*Background group*。**Android平台**在这些**linux control group**上定义了执行约束，这样的话不同控制组会被分配不同的执行时间。 
当进程是*Foreground*或者*Visible*优先级，那么由该应用创建的所有线程都属于*Foreground group*。如果通过按下home键把应用放进后台，该应用的所有线程都在*Background group*。
但是仍然具有一些问题：当*Foreground*运行的应用开启很多后台线程时，这些线程默认与UI线程具有**相同的**priority和control group。因此仍然会影响**UI thread**的响应。为了解决这个问题，可以通过**background thread**与*foreground control group*的解耦达到，让其归为*background control group*。
通过`Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND)`**不止会降低其优先级**还会**保证该线程从该应用所在的控制组中脱离**，到*background control group*。


#### Summary
可以看到通过设置*priority*对*CFS*的调度施加影响，这是**系统层面**的。那么通过对*Foreground control group*进行设置，这是**应用层面**。

UI线程是最重要的线程，但是相比于其他线程进行调度并不特殊。就需要依赖应用本身避免后台线程影响UI线程-通过**降低优先级**或者让不是很重要的后台线程在**background control group**里面执行