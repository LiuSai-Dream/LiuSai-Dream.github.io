---
layout: post
title: Efficient Andriod Threading - Chapter 4
category: Android
---




## 4. Thread Communication

Android中，线程通信的重要性通过平台特定的handler/looper机制得到体现。

- 单向的数据pipe
- 共享内存通信
- 使用*BlockingQueue*实现消费者-生产者模式
- 消息队列的操作
- 向UI线程中发送任务

### Pipes
Pipes属于`java.io`。Pipes为**同一进程**中的**两个线程**提供一个单向数据通道。
pipe本身是一个**环形的缓存**。只对相连接的两个线程开放，保证了线程安全。

其适用场景：两个长时间运行的任务，并且其中一个向另外一个连续的发送数据。例如从UI线程中offload任务到其他线程。

pipe可以传送二进制和字符数据。二进制数据使用`PipedOutputStream`和`PipedInputStream`，字符数据使用`PipedWriter`和`PipedReader`使用。pipe的生命周期起始于writer**或者**reader线程建立连接（单方面的），结束于连接的关闭（单方面的）。

#### Basis Pipe Use
默认的缓存大小是1024。
如果pipe满了，写操作会block。如果pipe空的，读操作会block（当在UI线程中使用时要注意是不能阻塞）。
另外当写数据完成时，可调用`flush()`，当缓冲区为空的时候，读操作会阻塞，并且调用`wait()`进行1s的等待。如果不调用`flush()`，就会真正的等待1s，否则reader就会立即得到通知，而不用等待1s。增强了信息传递的及时性。

### Shared Memory

对象的引用在线程的栈上存储，但是对象本身在shared memory（实际是heap）中。

### Singaling

通过`wait()/wait(timeout)`或者等价的`await()/await(timeout)`，*timeout*参数表明了在继续执行前等待多长时间。
以及`notify()notifyAll()`或者等价的`signal()/signalAll()`

### BlockingQueue

*signaling*是底层的，高配置的机制，可以用于多种场景，但是容易出现错误。

java平台在*signaling*的基础上构建了一个高层的抽象来解决线程间单向任意数据的传递。
其使用场景在于生产者和消费者中间是一个队列，该队列具有blocking行为。例如`java.util.concurrent.BlockingQueue`
该队列封装了list实现和信号功能。

> ps:感觉和Pipe差不太多，单向的，阻塞的。只不过更加抽象

### Android Message Passing

之前讨论的技术都属于java，但是UI线程不能阻塞， 因此上述的**线程间通信**（还有进程间通信）适用于work thread之间的通信。

Android对UI线程和work线程之间的通信有自己的一套机制。UI线程传递要处理的消息到后台线程。 该机制是非阻塞的。位于api的`android.os`包中，其涉及到的类：
![Thread communication](http://ww3.sinaimg.cn/mw690/63293ed1jw1f5ng0xnsd7j20gp0290sy.jpg)

- *android.os.Looper*：与**唯一一个** **消费者线程**相连接的**message dispatcher**
- *android.os.Handler*：处理**消费者线程**中的消息，并且在**生产者线程**中把消息插入消息队列。**一个looper可以有多个关联的handler**，但是多个handler把消息插入到同一个队列中。
- *android.os.MessageQueue*：在**消费者线程**中的消息队列，**一个looper**，**一个thread**，**只有一个MessageQueue**
- *android.os.Message*：在**消费者线程**中处理的消息。

**消费者线程** == **work thread**。

消息的处理过程：

1. 插入：**生产者线程**把消息插入到队列中，通过连接到**消费者线程**中的Handler来实现。
2. 检索：在消费端运行的Looper，从队列中取消息。
3. 分发：Handler负责在**消费者线程**处理消息。***一个线程**可以具有**多个handler实例**来处理消息。**Looper保证了消息分发到正确的Handler**。每个Handler处理其自己分发的消息。

### Classes Used in Message Passing

#### MessageQueue

使用`android.os.MessageQueue`类。使用单向链表实现。messages基于时间排序。 然而只有当事件的时间小于当前时间才被*dispatch*。否则，只有在`MessageQueue`继续*Pending*。
![message状态](http://ww2.sinaimg.cn/mw690/63293ed1jw1f5ngj4svutj20gn07hgm1.jpg)
从上图中可以看到观察到message的状态，所有的都处于*pending*状态，有一个越过了*dispatch barrier*，处于*dispatch*状态。

当没有*dispatch*状态的消息时，消费者线程就会阻塞。一旦有，消费者线程会立即执行。

生产者可以在**任何时间**把新的message插入到队列的**任何位置**。插入的位置基于时间值。如果新的message具有最小的时间值，那么它就会插入到队列的开头。

#### MessageQueue.IdleHandler

如果没有可处理的message，消费者线程就有了空闲时间。默认情况下，消费者线程在空闲的时候等待新的message。如下图：
![idle](http://ww2.sinaimg.cn/mw690/63293ed1jw1f5nh02ucg9j20gu082gly.jpg)

然而消费者线程可以利用空闲时间处理其他的任务。

通过实现`MessageQueue.IdleHandler`接口，可以让消费者线程在空闲的时候执行该接口的内容。可通过如下方法设置：
```
// Get the message queue of the current thread
MessageQueue mq = Looper.myQueue();

// Create and register an idle listener
MessageQueue.IdleHandler idleHandler = new MessageQueue.IdleHandler();
mq.addIdleHandler(idleHandler);

// Unregister an idle listener
mq.removeIdleHandler(idleHandler);
```
其接口为：
```
interface IdleHandler {
    boolean queueIdle();
}
```
当messagequeue检测到空闲时间时， 会调用所有注册（是如何执行的？实际上是顺序执行？）的`IdleHandler`。
此外应该避免在`queueIdle()`中执行长时间的操作，因为这样会延时后面的message的处理。

`queueIdle()`返回boolean值：

- *true* : 该idle handler继续存活，会在后续的空闲时间内接受调用

- *false*： 该idle handler不再存活。在后续的空闲时间段内不再接受调用。相当于使用`MessageQueue.removeIdleHandler()`移除listener。

#### Message
在`MessageQueue`中的元素是`android.os.Message`类。其为一个容器对象，可以存放**数据项**或者**任务**。

> 注意：message 知道它相关联的Handler，可以通过`Message.sendToTarget()`入队自己。例如：
`Message m = Message.obtain(handler, runnable);` ， `m.sendToTarget();`

##### Data message
其具有的字段：

- `what` (int)：message 标识符；
- `arg1, arg2` (int)：简单的数据类型；
- `obj`（object）：任意对象，如果需要跨进程，那么该对象需要实现*Pracelable*；
- `data`（Bundle）：Bunlder，任意数据类型的容器；
- `replyTo`（Messager）：**其他进程Handler的引用**，能够进行**双向**通信；
- `callback`（Runnable）：在thread上执行的任务。

##### Task message
使用`java.lang.Runnable`对象来表示，在消费者线程上执行。Task message不能包含有除了task本身的任何数据。

一个`MessageQueue`中可以含有data和task messages，以顺序的方式来执行。

Message的生命周期可分为四个阶段：

![message liefcycle](http://ww3.sinaimg.cn/mw690/63293ed1jw1f5nhhob19lj20gs02vweo.jpg)

*runtime*在**应用的message pool中**存储message对象。避免了创建新实例的开销。
上述的状态转换部分依赖应用，部分依赖平台。注意，上面的状态是无法观察到的。

###### Initialize
应用创建message对象，从对象池中获取一个对象：

- Explicit object construction
    `Message m = new Message()`
- Factory methods：
    - Empty message:
        `Message m = Message.obtain();`
    - Data message:
        ```
        Message m = Message.obtain(Handler h);
        Message m - Message.obtain(Handler h, int what);
        Message m - Message.obtain(Handler h, int what, Object o);
        Message m - Message.obtain(Handler h, int what, int arg1, int arg2);
        Message m - Message.obtain(Handler h, int what, int arg1, int arg2, Object o);
        ```
    - Task message:
    `Message m = Message.obtain(Handler h, Runnable task);`
    - Copy constructor:
    `Message m = Message.obtain(Message originalMsg);`
    
###### Pending
message被插入到了队列，等待被*dispatch*

###### Dispatched
`Looper`把message**从队列中取走**。该message被分配到消费者线程且**被执行中**。该状态由`Looper`来控制，并且负责把该message发送到正确的接收者。

###### Recycled
message状态被清空，实例回到**message pool**。由Looper负责回收message。回收的操作由*runntime*进行，而不是应用进行。

> 注意：一旦message被压入队列，其内容不应该被改变。理论上，在*dispatch*之前都可以更改message的内容。然后由于无法获得message的当前状态。有可能此时消费者线程正在处理该message。更加糟糕的是message已被回收并且被重复使用。
    
#### Looper

为所在的线程运行一个message loop。普通的线程并没有与之相关的loop，在线程中调用`prepare()`创建一个loop，然后调用`loop()`来处理message，直到loop被停止。
```
class ConsumerThread extends Thread {
    public void run() {
        Looper.prepare();
        
        // Handler creation omitted
        
        Looper.loop();
    }
}

```

1. 使用静态方法`prepare()`创建`Looper`，它会创建与当前线程相关的message queue。 此时，可以向其中插入message，但是无法dispatch message。
2. `loop()`是一个阻塞操作，保证了`run()`方法不会停止。此时可以dispatch message到消费者线程中进行处理。

需要长时间运行的message由于会阻塞队列中后续的message，因此不应该使用。

###### Looper termination
可通过`quit`或者`quitSafely`来停止处理消息。`quit()`停止*message queue*中的所有消息，包括*pending*，*dispatch*状态的（正在执行的不受影响？）。 `quitSafely`会继续执行*dispatch*的message（pass the dispatch barrier）。

停止`Looper`并没有停止`Thread`。其只是退出了`Looper.loop()`，从而让thread顺着其逻辑继续执行。
但是停止之后，就不能在线程中继续使用之前的`Looper`或者创建一个新的。所以，该线程无法继续处理message。如果仍然调用`Looper.prepare()`，会抛出`RuntimeException`。如果调用`Looper.loop()`，会阻塞，但是不会dispatch message。


##### The UI thead Looper

UI线程是唯一个默认与`Looper`关联的线程。它是一个普通的线程。在*application*被初始化之前，*UI thread*就与*Looper*进行了绑定。

UI线程的Looper与其他线程的Looper的差别在于：

- 可以通过`Looper.getMainLooper()`方法在任何地方获取
- 它无法被终止，调用`Looper.quit()`会抛出`RuntimeException`异常
- *runtime*通过调用`Looper.prepareMainLooper()`来关联Looper到UI线程。因此后续人为绑定新的Looper到UI线程会抛出一个异常

#### Handler

Handler既负责在**生产者线程**把message插入队列，同时负责在**消费者线程**处理message。

其作用：

- 创建消息
- 插入消息到队列
- 在消费者线程中处理消息
- 管理队列中的消息

##### Setup
`Handler`直接相关`Looper`。没有`Looper`，`Handler`无法进行工作。因此`Handler`实例需要在构造的时候与`Looper`进行绑定。

- 隐式的与**当前线程**的Looper绑定
    ```
    new Handler();
    new Handler(Handler.Callback);
    ```
- 显式的进行绑定
    ```
    new Handler(Looper);
    new Handler(Looper, Handler.Callback);
    ```

如果当前线程没有`Looper`（例如还没有调用`Looper.prepare()`），并且使用隐式的方式进行构建`Handler`。那么隐式绑定无法完成绑定，抛出`RuntimeException`。

一个thread可以有多个handler。
![handler和message的关系](http://ww2.sinaimg.cn/mw690/63293ed1jw1f5nijgnnolj20gr052aak.jpg)
> 注意：多个handler并不能并行执行。所有的message仍然在同一个messagequeue中，按顺序执行。

##### Message creation

为了简化，Handler类提供了对象的工厂函数来创建Message类：
```
Message obtainMessage(int what, int arg1, int arg2);
Message obtainMessage();
Message obtainMessage(int what, int arg1, int arg2, Object obj);
Message obtainMessage(int what);
Message obtainMessage(int what, Object obj);
```

从handler中得到的message是从message pool中获取的，并且隐式的与该handler实例绑定。

##### Message insertion
根据message的类型，Handler可以使用不同的方式把message插入到队列中。*Task message*通过前缀`post`方法来插入，*Data message*通过前缀`send`方法来插入。

- Add a task to the message queue:
    ```
    boolean post(Runnable r) // Runnable被封装为message
    
    ...
    ```
- Add a data object to the message queue:
    ```
    boolean sendMessage(Message msg)
    
    ...
    ```
- Add Simple data object to the message queue:
    ```
    boolean sendEmptyMessage(int what) // what被封装为message
    
    ...
    ```
    
每个message都有一个时间参数：

- default： Immediately eligible for dispatch
- at_front：eligible for dispatch at time 0. Hence, It will be the next dispatched message, unless another is inserted at the front before this one is processed
- delay：the amount of time after which this message is eligible for dispatch
- uptime：the absolute time at which this message is eligible for dispatch

尽管可以指定*delay*和*uptime*，但是实际的执行时间仍然是不确定的，由于message的处理是顺序的，依赖于已有的message的执行进度和操作系统调度。

插入操作并不是失败安全的，其可能遇到的错误：
| Failure    | Error response    | Typical application problem    |
| ------ | :------: | :-----: |
| Message has no Handler  | RuntimeException | Message was created from a Message.obtain() method without a specifed Handler |
|Message has already been dispatched and is being processed  | RuntimeException | The same message instance was inserted twice |
| Looper has exited  |  return false |  Message is inserted after Looper.quit() has been called |
    
> 注意：Handler类的`dispatchMessage`方法被`Looper`使用
，如果应用直接使用，那么该message会在调用线程（生产者线程）上直接执行，而不是消费者线程上执行。

##### Message processing

Message的type决定了其处理方式：

- *Task messages*： Task message 只包含一个*Runnable*，不包括数据。因此处理的过程定义在*Runnable*的run方法中，**该run方法自动在消费者线程上执行，而不通过`Handler.handleMessage()`方法**
- *Data Message*：message含有数据，Handler接收数据并处理。消费者线程通过`Handler.handleMessage(Message msg)`方法来处理。有两种方式来处理：1. 在创建Handler的时候定义`handleMessage`。该方法必须在message queue（在`Looper.prepare()`之后）可用的时候（在`Looper.loop()`之前）就要定义到（主要是要抓住，Looper/MessageQueue**产生**，MessageQueue**能够接受数据**之间的这个空隙把处理函数给定义好）。其模板如下：
```
class ConsumerThread extends Thread{
    Handler mHandler;
    public void run() {
        Looper.prepare();
        mHandler = new Handler(){
            public void handleMessage(Message msg) {
            }
        };
        Looper.loop();
    }
}
```
2. 另外一种方法就是使用`Handler.Callback`接口来扩展`Handler`类。
```
public interface Callback {
    public boolean handleMessage(Message msg);
}
```
有了这个接口，就没有必要继承`Handler`类，相反，可以把`Callback`的实现传递给`Handler`构造函数，然后就能接受和分发消息处理：
```
public class HandlerCallbackActivity extends Activity implements Handler.Callback {

    Handler mUiHandler;
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstancesState);
        mUiHandler = new Handler(this);
    }
    
    public void handleMessage(Message msg) {
        
        return true;
    }

}

```

如果该message已经处理完成，`Callback.HandleMessage`应该返回true，这样后续不再处理该message。如果返回false，那么该message会继续传递到`Hanlder.handleMessage`处理。

注意`Callback`没有override `Hanlder.handleMessage`，它只是对message的**预处理**。`Callback`预处理能够在`Handler`处理它们之前**中断和改变**message。下面示例了如何使用`Callback`来中断message。
```
public class HandlerCallbackActivity extends Activity implements Handler.Callback {
    
    public boolean handleMessage (Message msg) {
        switch(msg.what) {
        case 1:
        msg.what = 11;
        return true;
        
        case 2:
        msg.what = 22;
        return false;
        }
    
    }
    
    // 还需要这个函数？？？  对于Callback接口和Handler类的结合使用不是很清楚
    public void onHandleCallback(View v) {
        Handler handler = new Handler(this) {
            public void handleMessage(Message msg) {
                
            }
        } ;
        
        handler.sendEmptyMessage(1);
        handler.sendEmptyMessage(2);
        
    }
    
}

```

#### Removing Messages from the Queue

在message入队之后，生产者可以移除该message，只要没有被*Looper*出队。有时需要把队列情况，有时只需要移除几个message。
因此需要定位到这些message。

定位的方式：

| Identifier type | Description | Messages to which it applies|
| :----- | :------- | :---------- |
|Handler（范围最大） |  Message receiver | Both task and data message|
|Object（某一类型）  |  Message tag |  Both task and message|
|Integer（某一类型） |  what parameter of message | Data messages|
|Runnable （某一个）| Task to be executed  | Task messages |

*Handler identifier* 对于每个message是强制性的，因为message总是知道会被dispatch到那个Handler。因此一个Handler无法去除由其他Hadnler发送的message。
对应的方法：

- remove task message
```
removeCallbacks(Runnable r)
removeCallbacks(Runnable r, Object token)
```
- remove data message
```
removeMessages(int what)
removeMessages(int what, Object o)
```
- remove task and data message
```
removeCallbacksAndMessages(Object token)
```

`Object`在data和task message中都存在，因此可以作为tag使用。

但是一旦message被分发和处理了就不能移除了。但是并没有方式知道message是否已经被处理了。

#### Observing the Message Queue
可以通过与Handler相关联的Looper来观察pending message 以及 dispatching message。Android平台提供了两个机制。

1. Taking a snapshot of the current message queue
`mWorkerHandler.dump(new LogPrinter(Log.DEBUG, TAG)," ")`
2. Tracing the message queue processing
message的处理信息可以打印到log。可以从`Looper`类中开启消息队列logging。`Looper.myLooper().setMessageLogging(new LogPrinter(Log.DEBUG, TAG))`

### Communicating with the UI Thread

UI线程可以当作一个消费者，其他的线程可以作为生产者，但是此时任务不能过于繁重。

通过全局可以获取的`Looper`来向UI线程发送message。 （**注意这种方法只能够处理task message，不能够处理data message**）
```
Runnable task = new Runnable(){ };
new Handler(Looper.getMainLooper()).post(task);
```
当然UI线程也可以向自己发送消息，此时该message会**被很快的处理**。例如：
```
// Method called on UI thread
private void postFromUiThreadToUiThread () {
    new Handler().post(new Runnable() { ... }  );
    // The Code at this point is part of a message being processed
    // and is executed **before the posted message**也就是说上面那句尽管调用了post，但是**并不会立即执行**，因为此处的代码会比上面的执行更早
}
```
然而，在UI线程中发送的**task message**，可以**不通过传递直接由UI线程处理**，通过`Activity.runOnUiThread(Runnable)`，例如：
```
// Method called on UI Thread
private void postFromUiThreadToUiThread() {
    runOnUiThread(new Runnable() { ... } );
    
    // The code at this point is executed after the message。也就说上面那句会**立即执行**
}

```
**如果在UI以外的线程调用，该message会插入到队列中**（前面举例是在UI线程内调用），**`runOnUiThread`方法只能在`Activity`实例上执行**。但是该行为可以通过跟踪UI线程的ID同样能实现。例如：
```
public class EatApplication extends Application {
    private long mUiThreadId;
    private Handler mUiHandler;
    
    public void onCreate() {
        super.onCreate();
        mUiThread = Thread.currentThread().getId();
        mUiHandler = new Handler();
    }
    
    public void customRunOnUiThread(Runnable action) {
        if (Thread.currentThread().getId() != mUiThreadId) {
        mUiHander.post(action);
        } else {
            action.run();
        }
    }
}
```
然而这种方法由于无法访问到activity中的UI部件，仍然不能更新UI，只是在UI线程上执行，意义不大。

