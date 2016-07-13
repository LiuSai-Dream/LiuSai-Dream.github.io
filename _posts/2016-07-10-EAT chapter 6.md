---
layout: post
title: Efficient Andriod Threading - Chapter 6
category: Android
---



## 6. Memory Management

介绍了*thread*和*thread communication*是如何导致内存泄漏的。
并且提供了正确设计和生命周期管理的策略。

### Garbage Collection

每个应用有自己的Dalvik VM和它自己的垃圾回收策略。（垃圾回收是局限于单个app的，而系统杀进程是全局的）。

直到Gingerbread之前，gc与应用（在主线程上）顺序执行，当gc的时候应用会出现暂停。
从Honeycomb之后，gc在自己的线程中执行，不会再暂停应用的线程。

Dalvik gc使用的是**标记**和**清除**。 首先标记出不再使用的对象，清除阶段把标记的对象清除。

一个对象只有不被gc root遍历到，才会被认为不被用到。但是gc root本身不会回收掉，尽管没有其他的对象引用它。

任意在**堆（heap）之外**的对象都被认为gc root。包括**静态对象**、**栈上的局部对象**、和**线程**。因此**线程中直接或者间接引用**的对象在**线程执行**的时候都会造成内存泄漏 （Any object that is accessible from outside the heap is considered to be a GC root. this includes static object, local object on the stack, and threads. Thus, object directly or indirectly referenced from a thread will be reachable during the execution of the thread）


### Thread-Related Memory Leaks

在线程中引用的对象在**线程运行期间**无法被释放，只有当线程**执行完成**之后才能够被回收。

线程分为：**UI 线程**、 **binder 线程**、 **worker 线程**。 后面的两种线程可以在应用执行期间被开启和终止，其执行的期间可能会造成内存泄漏。

可能泄漏内存的线程有两种特征：

- *Potential risk*
    内存泄漏的风险随着线程**运行的时间**越长越大。生命短的线程造成泄漏的风险不大。

- *Leak size*
    如果泄漏的**内存比较大**，例如bitmaps，view hierarchies。可能会占满内存。


### Thread Execution

![thread lifecycle of gc root](http://ww4.sinaimg.cn/mw690/63293ed1jw1f5p3pfvhbxj20go07gt93.jpg)

worker thread的生命周期通常包含三步：

1. 创建thread对象
2. 开启thread / Runnable执行，此时*Thread/Runnable*对象本身就是GC root，它所引用的所有对象都是可达的。
3. 终止thread

在**方法**中创建的对象只有方法运行完成后才会被回收，除非方法**返回该对象**，**被其他对象引用**。

下面列举一些线程定义例子，展示了一些对象树。

#### Inner classes

内部类是外部类的成员，具有访问外部类其他成员的能力。因此**内部类隐式的持有一个外部类的引用**。
那么使用内部类来定义的线程，其会持有外部类的引用，外部类在内部类线程运行的时候就不会被回收。
![tree of inner class](http://ww3.sinaimg.cn/mw690/63293ed1jw1f5p406bo6gj20gq05s0sv.jpg)

示例：

```
public class Outer {

    public void sampleMethod() {
    
        SampleThread sampleThread = new SampleThread();
        sampleThread.start();
    }
    
    private class SampleThread extends Thread {
        public void run() {
            Object sampleObject = new Object();
        }
    }
    
}

```

#### Static inner classes

静态内部类是外部对象**class**实例的成员。静态内部类持有外部对象**class**的引用，但是与外部类对象无关。（此处是**class**而不是类实例）
![static inner class](http://ww1.sinaimg.cn/mw690/63293ed1jw1f5p43arxfhj20gq05odfz.jpg)


```
public class Outer {
    
    public void sampleMethod() {
        SampleThrad sampleThread = new SampleThread();
        sampleThread.start();
    }

    private static class SampleThread extends Thread {
        
        public void run() {
            Object sampleObject = new Object();
        }
    
    }

}

```

然而有些时候，程序员想要把执行环境（Thread）和任务（Runnable）分开。如果把*Runnable*作为内部类，那么*Runnable*在执行的时候会持有外部类的引用，尽管由static  inner class执行。
![runnable with static inner class](http://ww1.sinaimg.cn/mw690/63293ed1jw1f5p4ak3sjoj20go05qdg1.jpg)

> 可以看到，尽管使用了static inner class，但是仍然持有**Outer 实例引用**以及**Outer class实例引用**。 

示例：

```
public class Outer {
    
    public void sampleMethod () {
        SampleThread st = new SampleThread(new Runnable() { // 相比于上面的两个例子，此处使用了Runnable作为执行体，而不是Thread本身
            public void run() {
                Object so = new Object();
            }
        });
        st.start();
    }

    private static class SampleThread extends Thread {
        
        public SampleThread (Runnable runnable) {
            super(runnable);
        }
    
    }

}

```

#### The lifrcycle mismatch

Android平台发生泄漏的原因存在于部件、对象、线程之间生命周期的不匹配。

Android不止处理对象的生命周期，也处理部件的生命周期。**四大组件的生命周期并不完全与其对象的生命周期一致**。
![component and instance](http://ww1.sinaimg.cn/mw690/63293ed1jw1f5p4if4tkfj20gr06774q.jpg)

***Activity*对象的泄漏是最严重的部件泄漏**。*Activity*持有的view hierarchy包含了大量的堆内存。

*Activity*的`onCreate()`方法的执行，导致了*Activity*对象的创建。从`onCreate()`到`onDestroy()`该对象一直被使用。当***Activity对象*finish**或者**被系统destroy**（注意这两者的区别）。
部件生命周期以调用`onDestroy`结束（也就意味着当该函数被调用的时候，部件对象就被销毁了？），其发生的场景有：

- 用户按下了back键。隐式的结束了*Activity*
- *Activity*显式的调用`finish()`
- 系统杀掉*Activity*所在的进程
- 配置改变，设备旋转，销毁*Activity*并且创建一个新的

![activity ](http://ww1.sinaimg.cn/mw690/63293ed1jw1f5p4mnz45bj20gv068t90.jpg)

多个同一*Activity*的对象可以同时存在。并且由于*thread*对它的引用，导致一直无法释放。

### Thread Communication

不止线程的执行，线程之间的通信也是一种潜在的内存泄漏。
而**线程执行**的泄漏多发生在worker线程中，而**线程通信**则可以发生在UI线程中。
对于接收message的线程，其会有个***MessageQueue*来存放*pending message***，有个***looper*用来分发*message***，有个***handler*来处理*message***。 大多数对象被**生产者线程**所引用，当线程退出的时候，这些对象就会gc，但是**handler**由于**被消费者线程**通过**对象链引用**，可能会导致内存泄漏。

![receive and reference objects](http://ww1.sinaimg.cn/mw690/63293ed1jw1f5p4xew7f4j20go05bt8z.jpg)

当生产者运行的时候，在thread之间传递的message持有handler的引用，以及Object(data message)或者Runnable(task message)引用。 从message刚创建，该message持有消费者线程的引用。当message在message queue中或者在thread执行的时候，不能够进行gc。 

#### Sending a data message
*Outer*类持有*Handler*，并且*Handler*与创建*Outer*类的线程相连接：

```
public class Outer {
    Handler handler = new Handler() {
        public void hadnlerMessage(Message msg) {
            // Handle message
        }
    };
    
    public void doSend() {
        Message msg = mHandler.obtainMessage();
        msg.obj = new SampleObject();
        mHandler.sendMessageDelayed(msg, 60*1000);
    }
}

```

![object reference when send data message](http://ww2.sinaimg.cn/mw690/63293ed1jw1f5p5c8sp4pj20gu074weo.jpg)

#### Posting a task message
使用*Runnable*相比于*Thread*，具有额外的*Outer*类的引用。

```
public class Outer {
    Handler mHandler = new Handler() {
        public void handleMessage(Message msg) {
        
        }
    };
    
    public void doPost() {
        mHandler.post(new Runnable() {
            public void run() {
            
            }
        });
    }
    
}

```
![object reference when posting task message](http://ww2.sinaimg.cn/mw690/63293ed1jw1f5p5gabl0lj20gr07hmxg.jpg)


### Avoiding Memory Leaks

#### Use Static Inner Classes
局部类(Local class)，内部类(inner class)，匿名内部类(anonymous inner class)都隐式的持有外部类的引用。因此泄漏的不止是它们自己，也包括被外部类引用的内容。例如，Activity和它的view hierarchy。

与其使用嵌入内部类，还不如使用静态内部类，因此静态内部类引用的是全局**class对象**，而**不是实例对象**。 


#### Use Weak References within static inner class

静态内部类无法访问外部类的实例变量。为了需要访问外部类的实例变量，可以使用`java.lang.ref.WeakReference`来引用外部变量：

```
public class Outer {

    pirvate int mFiled;
    private static class SampleThread extends Thread {
        private final WeakReference<Outer> mOuter;
        SampleThread(Outer outer) {
            mOuter = new WeakReference<Outer> (outer);
        }
        
        public void run() {
        // Check for null as the outer instance may have been GC'd
        
            if (mOuter.get() != null) {
                mOuter.get().mField = 1;
            }
        
        }
        
    }
}

```

在静态内部类中使用*weak reference*来引用外部类，但是*weak reference*并不是gc引用计数的一部分。

#### Stop Worker Thread Execution

当不需要继续执行的时候把线程停掉。

#### Retain Worker Threads

保持旧的线程，给新的activity使用，从旧的activity中移除线程的引用。

#### Clean Up the message Queue especially UI thread （前面几个与**线程执行**相关，这个与**线程通信**相关）

如果*MessageQueue*中的*Message*不需要处理，就移除。

在*message queue*中的*message*一旦当*worker thread*执行完成后就可以被gc（也就是说可以直接结束该线程而不用先清理*message queue*）。但是*UI thread*会到进程完成后才会退出。因此**清除*UI thread*中的*message*对预防内存泄漏很重要**。