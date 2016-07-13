---
layout: post
title: Efficient Andriod Threading - Chapter 7
category: Android
---


## 7. Managing the Lifecycle of a Basic Thread

包括任务的取消，跨activity和fragment保存线程。

### Basics
*Android* 和 *java* 中的*Thread*是一样的。最接近Linux原生的线程。
*Thread类*通过Runnable为任务创建执行环境。
*Thread*类实现*Runnable*接口，任务的执行要么由*thread*自己定义，或者在线程创建的时候注入。

#### LifeCycle

线程可能的状态在`Thread.State`类中定义：
![lifecycle of thread](http://ww3.sinaimg.cn/mw690/63293ed1jw1f5q7l1qpd3j20gp0413yj.jpg)

- *New* 在执行之前。该实例**并没有设置执行环境**，因此该线程并不比其他的实例对象要笨重。默认，构造器把新创建的线程赋予**创建者线程**所在的线程组，具有相同的优先级，从UI创建的线程属于相同的线程组，与UI具有相同的优先级。

- *Runnable* 在`Thread.start()`调用之后，**设置了执行环境**。现在是可运行的状态，当调度器选择该线程执行的时候，会调用`run`方法。

- *Blocked/Waitting* 阻塞状态。显式的使用：1. *Thread.sleep()*：线程sleep特定的时间，但是不释放持有的锁。2. *Thread.yield()*：放弃执行，让调度者选择下一个要执行的进程，但是下一个调度的也可能是该线程本身，也不会释放持有的锁。
两者的区别：`sleep()`方法会给其他线程运行的机会，**不考虑其他线程的优先级**，因此会给较低优先级线程一个运行的机会，并且当前线程将转到**阻塞状态**； `yield()`只会给**相同优先级或者更高优先级**的线程一个运行的机会。当前优先级会转到**就绪状态**。

- *Terminated*：当`run`方法返回后，线程的最终状态，无法重用**Thread实例**或者它的**执行环境**。Setting up和tearing down执行环境是**heavy operation**，如果频繁的执行这些操作，就要使用线程池。

#### Interruptions

有时，应用希望中断线程的执行。然而无法直接中断。但是线程是有*interruptd*的，该函数只是**请求**线程*terminate*，但是线程可以**不遵守**。该函数只是在*thread*内部设置一个标识，表示其为interrupted状态。线程必须自己**检查该标识**，以便更加优雅的退出，例如：

```
public class SimpleThread extends Thread {

    public void run() {
        while(isInterrupted() == false) {
            
        }
        
        // Task finished and thread termiates
    }
}

```

该interrupt flag同样被许多阻塞方法和库支持，一个当前**被阻塞的线程**在被*interrupted*的时候会抛出*InterruptedException*，线程可以在*catch*语句块中**清除资源**。当*InterruptedException*被抛出之后，*interrupted flag*会被**重置**，这可能会带来问题。
一些处理方法：

```
void myMethod() {
    try {
        // some blocking call
    } catch (InterruptedException e) {
        // 1. Clean up
        // 2. Interrupt again
        Thread.currentThread().interrupt();
    }
    
}

```

> 注意：Interruption状态可以使用静态方法`Thread.interrupted()`来检测。其返回值和`isInterrupted()`相同，然而`Thread.interrupted()`方法有个副作用：会像阻塞情况下抛出`InterruptedException`异常一样清除中断标志。

> 注意：不要使用`Thread.stop()`来中断一个执行的线程，这样会导致共享的数据处于一种不一致和无法预测的状态。

#### Uncaught Exceptions

如果在线程执行的过程中，未预期的错误发生了，结果就是抛出*unchecked*的异常。*unchecked* 异常是**RuntimeException**的子类，并且不需要强制*try/catch*进行捕获。因此该异常会顺着调用栈传递。
为了避免这种没注意到的异常错误，线程可以与一个`UncaughtExceptionHandler`相连接，其会在**线程中止之前**调用。该*handler*提供了一个优雅的停止线程的方式。或者通过网络或者文件通知该错误。

使用`UncaughtExceptionHandler`接口的方式是实现`uncaughtException`方法，并把它附加到线程上。如果线程由于未知的错误被终止了，那么在线程真正终止前会调用`uncaughtException`方法。
`UncaughtExceptionHandler`接口可以附件到所有的线程或者个特定的线程中。

*Thread global handler*

```
static void setDefaultUncaughtExceptionHandler( Thread.UncaughtExceptionHandler handler );
```

*Thread local handler*

```
void setUncaughtExceptionHandler(Thread.UncaughtExceptionHandler handler);
```

如果一个线程既设置了*global*也设置了*local*，那么只会执行*local*。

既可以通过`Thread.currentThread()`在任务执行的时候设置`UncaughtExceptionHandler`也可以通过线程本身的引用通过`setUncaughtExceptionHandler()`进行设置。

##### Unhandled Exceptions on the UI Thread

Android 运行时在应用启动的时候，会附件一个进程全局**UncaughtExceptionHandler**。该*exception handler*会附加到所有的线程中，并且**任何一个线程**发生*unhandled exception* 的时候，其结果是：**该进程被杀掉**。

默认的行为可以在全局上对所有线程进行更改也可以在特定的线程上进行更改。一般情况下，首先通过重写默认的运行时行为，然后把*exception*重定向到*runtime handler*:

```
// Set new global handler
Thread.setDefaultUncaughtExceptionHandler(new ErrorReportExceptionHandler());
// Error handler that redirects exception to the system default handler.
public class ErrorReportExceptionHandler
implements Thread.UncaughtExceptionHandler {
    private final Thread.UncaughtExceptionHandler defaultHandler;
    public ErrorReportExceptionHandler() {
    this.defaultHandler = Thread.getDefaultUncaughtExceptionHandler();
}
@Override
public void uncaughtException(Thread thread, Throwable throwable) {
    reportErrorToFile(throwable);
    defaultHandler.uncaughtException(thread, throwable);
}
}

```

#### How uncaught exceptions are handled

默认情况下，当异常没有被`try/catch`捕获的时候，java本身会打印出异常栈信息，然后结束程序。

实际上，java使用*uncaught exceptions*处理未捕获的异常。当*uncaught exception*在特定的线程中出现的时候，java会查找*uncaught exception handler*，接口*UncaughtExceptionHandler*的实现，该接口有个`handleException()`方法。那么我们就可以使用自己的`UncaughtExceptionHandler`来处理特定thread的uncaught exception。

当uncaught exception发生的时候，jvm的执行流程：

- 在异常发生的`Thread`类上调用一个私有的特殊方法，`dispatchUncaughtException()`
- 终止发生异常的线程

`dispatchUncaughtException`方法调用线程的`getUncaughtExceptionHandler()`方法，查找适合的uncaught异常处理方法。通常找到的是父线程的`ThreadGroup`，其默认的`handleException`默认情况下打印堆栈信息。

`setUncaughtExceptionHandler`通常用来释放内存，杀掉线程。





### Thread Management

通过对线程生命周期的三个阶段进行把控：*definition and start*, *retention* and *cancellation*.

#### Definition and Start
下面列举了最常用的定义和开启*worker thread*的方式。

##### Anonymous inner class

```
public class AnyObject {
    public void anyMethod() {
        new Thread() {
            public void run() {
                doLongRunningTask();
            }
        }.start();
    }
}
```

匿名内部类实现简单，但是其会持有外部类的引用。

##### Public thread

通过独立的类来定义线程，而不是在运行该线程的类里面定义：

```
class MyThread extends Thread {
    public void run() {
        doLongRunningTask();
    }
}

public class AnyObject {
    private MyThread mMyThread;
    
    public void anyMethod() {
        mMyThread = new MyThread();
        mMyThread.start();
    }
}

```

独立的类不持有外部类的引用。

##### Static inner class thread definition

```
public class AnyObject {
    static class MyThread extends Thread() {
        public void run() {
            doLongRunningTask();
        }
    };
    
    private MyThread mMyThread;
    
    public void anyMethod() {
        mMyThread = new MyThread();
        mMyThread.start();
    }
}

```

静态内部类持有外部类的*class*实例，而不是类实例。

##### Summary of options for therad definition

**内部类**持有外部类实例的引用，可能会导致较大内存的泄漏。
**匿名内部类**因为没有该线程的引用，无法控制这种线程。
此外要控制线程的个数，可以把线程放进一个链表里，或者只有当前面的线程被销毁的时候才创建新的线程。

线程池和*HandlerThread*对线程执行的个数做了限制。

#### Retention

在配置改变的情况下，处理线程比较好的方式是保留原有的线程，并且让新的*Activity*对象持有旧的*Activity*所开启的线程。

##### Retaining a thread in an Activity

*Activity*类包含了两个方法处理*thread retention*：

`public Object onRetainNonConfigurationInstance()` 在configuration变化之前被系统调用。导致当前的*Activity*被销毁。该函数的实现应该返回希望在配置改变后仍然保存的任何对象（例如：线程）并返回到新的*Activity*对象中。

`public Object getLastNonConfigurationInstance()`
在新的*Activity*中从`onRetainNonConfigurationInstance`方法中得到对象。可以在`onCreate()`或者`onStart()`中调用，如果*Activity*不是因为配置改变，返回的值为null。

下面的代码演示了因为配置改变在*activity*之间传递*thread*。

```
public class ThreadRetainActivity extends Activity {
    
    private static class MyThread extends Thread {
    
        private ThreadRetainActivity activty;  // 保留对象的印象的引用，会造成该对象无法被回收
        
        public MyThread(ThreadRetainActivity activity) {
    mActivity = activity;
}

    public void attach(ThreadRetainActivity activity) {
    mActivity = activity;
    }

    @Override
    public void run() {
        final String text = getTextFromNetwork();
        mActivity.setText(text);
    }
    
        private String getTextFromNetwork() {
    // Simulate network operation
    SystemClock.sleep(5000);
    return "Text from network";
    }
    
    }
    
    private static MyThread t;
    private TextView textView;
    
    @Override
public void onCreate(Bundle savedInstanceState) {
super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_retain_thread);
    textView = (TextView) findViewById(R.id.text_retain);
    Object retainedObject = getLastNonConfigurationInstance();
    if (retainedObject != null) {
    t = (MyThread) retainedObject;
    t.attach(this);
}
    
    @Override
public Object onRetainNonConfigurationInstance() {
    if (t != null && t.isAlive()) {
    return t;
    }
    return null;
    }
    public void onClickStartThread(View v) {
    t = new MyThread(this);
    t.start();
    }
    private void setText(final String text) {
    runOnUiThread(new Runnable() {
    @Override
    public void run() {
    textView.setText(text);
    }
    });
}
    
}

```

##### Retaining a thread in a Fragment
