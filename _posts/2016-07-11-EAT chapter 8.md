---
layout: post
title: Efficient Andriod Threading - Chapter 8
category: Android
---


## 8. HandlerThread: A High-Level Queueing Mechanism

`HandlerThread` 对线程的封装，自动的设置内部消息传递机制。

### Fundamentals

`HandlerThread`是一个具有*message queue*，*Thread*， *Looper*的线程。但是并没有用来处理消息的*handler*。它的构建和开启与普通的线程一样。

```
HandlerThread handlerThread = new HandlerThread("HandlerThread");
handlerThread.start();

mHandler = new Handler(handlerThread.getLooper()) {
    public void handleMessage(Message msg) {
    super.handlerMessage(msg);
    
    }
};

```

`HandlerThread`在内部设置*Looper*，并且接受*message*。内部设置保证了在创建*Looper*和发送消息之间没有竞争条件（如果手动的设置*Looper*，需要*Looper.prepare*和*Looper.Loop()*两步）。
平台通过调用`Thread.getLooper()`进行阻塞，直到`HandlerThread`准备接受消息来解决竞争条件。

如果在处理消息之前还需要额外的设置，那么应该重载`HandlerThread.onLooperPrepared()`,当*Looper*准备好之后在后台执行。 
应用可以在该函数中初始化代码，例如创建相关的*Handler*。

#### Limit Access to HandlerThread

通过继承HandlerThread，并且保持Handler私有来限制对其访问，并且保证了Looper也无法访问。 

```
public class MyHandlerThread extends HandlerThread {
    
    private Handler mHandler;
    
    public myHandlerThread() {
        super("MyHandlerThread", Process.THREAD_PRIORITY_BACKGROUND);
    }

    protected void onLooperPrepared() {
        super.onLooperPrepared();
        
        mHandler = new Handler(getLooper() {
            public void handleMessage(Message msg) {
                
            }
        
        });
    }
    
    public void publishedMethod1() {
        mHandler.sendEmptyMessage(1);
    }

    public void publishedMethod2() {
        mHandler.sendEmptyMessage(2);
    }
    
}

```

### LifeCycle

一个被终止的`HandlerThread`无法被重用。需要重新定义一个新的`HandlerThread`。
其生命周期包括的状态集合：

1. *Creation*：

```
HandlerThread(String name)
HandlerThread(String name, int priority)
```

其中的*priority*可以是`Process.THREAD_PRIORITY_BACKGROUND`

2. *Execution*：当处理消息的时候，`HandlerThread`是活动的（只要Looper可以dispatch消息到线程中）。该*分发机制*在线程启动的时候通过`HandlerThread.start`进行set up，在`HandlerThread.getLooper()`返回或调用`onLooperPrepared`的时候ready。`getLooper()`会阻塞直到`Looper`准备好。

3. *Reset*：*message queue*可以被重置，在queue中的消息被清空了，但是不会影响正在执行的消息。但是线程仍然是活跃的，并且可以处理新的消息。

4. *Termination*：通过`quit`或者`quitSafely`来终止`HandlerThread`,也就意味着`Looper`的终止。也可以发送*interrupt*到*HandlerThread*来取消当前处理的*message*。

```
public void stopHandlerThread(HandlerThread handlerThread) {
    handlerThread.quit();
    handlerThread.interrupt();
}
```

也可以通过发送一个*finalization*任务到*Handler*来*quit Looper*：

```
handler.post(new Runnable() {
    public void run() {
        Looper.myLooper().quit();
    }
});
```

可以看到`HandlerThread`的特性实际上是`Looper`，`MessageQueue`，`Handler`，以及`Thread`的结合。

### Use Cases

利用了`HandlerThread`的**顺序执行**和**可控的生命周期**特性。

#### Repated Task Execution

`HandlerThread`提供了一个有效的在后台顺序执行任务的方式。
例如可以在*activity*开启的时候打开`HandlerThread`，在*activity*关闭的时候关闭`HandlerThread`。能够与*activity*的生命周期类似。


#### Related Tasks

##### Example: Data persistence with SharedPreferences

`SharedPreferences`是在文件系统中持久化的手段。 但是系统访问文件并不是线程安全的。由于`HandlerThread`的顺序执行特性，不用增加异步支持，就可以直接用。

#### Task Chaining

##### Example: Chained networks calls

#### Conditional Task Insertion

`HandlerThread`能够很好的对*queue*中的*Message*进行控制。

```
handler.hasMessages(int what)
handler.hasMessages(int what, Object tag)
```

有条件的消息插入保证了不会再消息队列中插入已经有的消息。

```
if (handler.hasMessages(MESSAGE_WHAT) == false) {
    handler.sendEmptyMessage(MESSAGE_WHAT);
}
```
