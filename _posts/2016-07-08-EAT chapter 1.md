---
layout: post
title: Efficient Andriod Threading - Chapter 1
category: Android
---

## Android Components and the Need for Multiprocessing

首先介绍Android平台，应用架构，应用的执行。

### Android Software Stack
---

第三方应用运行在软件栈上。
![软件栈](http://ww3.sinaimg.cn/mw690/63293ed1jw1f5mtgf9ckdj20gm04x74k.jpg)

- *应用*：使用java开发，同时使用java库和Android框架库
- *Core Java*：*core java*库被应用和应用框架使用。它是*Apache Harmony*实现的子集，基于java5。 提供了`java.util.concurrent`和`java.lang.Thread`
- *Application framework*：定义并且管理Android 部件的生命周期和通信。 此外也定义了Android特有的异步机制。例如：`HandlerThread`, `AsyncTask`, `IntentService`, `AsyncQueryHandler`, `Loaders` （注意**通信**和**异步机制**是不一样的概念）
- *Native libraries*：C/C++类库处理图形，多媒体，数据库，字体等，java应用不直接与*native libraries*交互，而是通过*Application framework*对*native libraries*进行封装来使用
- *Runtime*：在虚拟机中的沙盒运行环境中执行编译后的代码。*Runtime*可以是*Dalvik*或者*ART*
- *linux kernel*：管理硬件，进程和线程。每个进程具有一个运行应用的运行时环境

### Application Architecture
---

应用的基石是`Application`，`Activity`，`Service`，`BroadcastReceiver`，`ContentProvider`

#### Application
使用`android.app.Application`对象表示执行的应用。当应用开启的时候初始化，应用停止的时候销毁。当进程被终止，重启的时候，一个**新的**`Application`对象会被创建。

#### Components

四大部件由`runtime`管理，都代表了应用的进入点，任何一个都可以进入应用。

*Components*及其生命周期是Android特有的，然而其生命周期并不底层的java对象一致。 一个java对象可以outlive它对应的component。 并且*runtime*可以多个java对象对应一个component。这种情况下要注意内存泄漏了。

##### Activity
包括UI部件并且持有包含所有`View`实例的*view hierarchy*，因此其内存占用可能会很大。

当按下*back*或者调用`finish()`时`Activity`生命周期结束。

##### ContentProvider

当需要为其他应用提供数据的时候，才考虑使用`ContentProvider`

##### BroadcastReceiver

动态注册：当希望监听时注册，停止监听时取消。
静态注册：在**应用安装期间**监听`intent`。

### Application Execution
linux kernel处理多任务，任务的执行基于linux进程。

#### Linux Process
linux为每个**用户**分配一个**user ID**，用来隔离用户。 每个用户的私有资源因为权限不会被其他用户得到。除非是root用户。
使用沙盒用来隔离用户。 
Android中每个**应用包**具有一个唯一的user ID（有可能多个应用包使用同一个 user ID，但是不会绝不会出现user ID重复现象）。例如，一个**应用**对应着一个**用户**。
![process、vm、app](http://ww1.sinaimg.cn/mw690/63293ed1jw1f5mumut7zyj20gn044q36.jpg)

默认情况下，应用和进程具有一对一关系。但是一个应用也可以运行在多个进程中。此外多个应用也可以运行在相同的进程中。

#### Lifecycle

应用的生命周期封装在linux进程中，在java层面，应用与`android.app.Application`类对应。当*runtime*调用`Application`的`onCreate()`方法时开启应用，理想情况下，当*runtime*调用`onTerminate()`方法时关闭应用。但是应用本身不应该依赖这种方式。
因为运行时环境的上层还有linux进程，而linux进程可能会随时被kill而来不及调用`onTerminate()`。 
`Application`对象在进程中是第一个被实例化，最后一个被销毁的对象。

##### Application start

当应用的任何一个组件被执行，开启该应用，并且创建linux进程。其开启顺序：

1. start linux process
2. create runtime （所有的app共用一个runtime还是一个app对应一个runtime）
3. create Application instance
4. create the entry point component for the application

开启linux进程和runtime并非一个实例化操作。它会降低性能影响用户体验。
因此系统会在开机的时候开启一个特殊的进程叫做*Zygote*，*Zygote*预先加载了*core libraries*，而core libraries是系统层面所有应用共享的，因此不需要每次都拷贝。 因此新的应用进程会从*Zygote*中进行fork。

##### Application termination
当系统希望free up资源的时候会结束应用。因此尽管一个应用的所有部件都被销毁，但是该应用本身并不一定被销毁。

进程的排序规则：

1. *Foreground*：在*front*具有*visible*部件；其他进程中的*service*与*front 的 activity*绑定；*BroadcastReceiver*正在执行
2. *Visible*：应用具有可见性部件，但是部分可见（与上面的区别是该部件并不*front*）
3. *Service*：后台执行，且没有和*visible*部件绑定
4. *Background*：具有*nonvisible*的*Activity*
5. *Empty*：没有任何活动部件的进程。保留空进程的目的是为了提升开启时间，但是当资源不足的时候它们是第一个被关闭的。

##### Lifecycles of Two Interacting Applications
需要注意的是，真实的应用生命周期（由linux进程表示）和用户感知的应用生命周期并不是严格一致的。 系统可能正在运行用户认为已经停止的应用。而这些应用是以空进程的形式存在的。这些空进程为了缩短启动时间而存在。

### Structing Application for Performance

#### Creating Responsive Applications Throuogh Threads

使用多个CPU能够提高吞吐量，但是并不能保证**响应**及时。UI的响应根据UI线程，然而只能有一个UI线程。

为了保证响应及时，不应该在UI线程中做一些长时间执行的工作。
长时间操作包括：

1. Network communication
2. Reading or writing to a file
3. Creating, deleting and updating elements in database
4. Reading or writing to SharedPreferences
5. Image processing
6. Text parsing

例如动画，1s更新60帧，每帧需要16ms。那么UI线程中的操作要在16ms内完成。

### Summary
所有部件上的执行都在UI线程上。
Android应用运行在Dalvik虚拟机中，Dalvik虚拟机在linux系统的进程上执行。