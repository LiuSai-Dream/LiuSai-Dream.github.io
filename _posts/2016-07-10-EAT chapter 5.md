---
layout: post
title: Efficient Andriod Threading - Chapter 5
category: Android
---



## 5. Interprocess Communication

上面说的是**线程间**通信，**进程间**的通信是通过*binder framework*来实现的，当线程之间没有共享的内存的时候，该框架管理数据的传输。

最平常的IPC使用是通过高层的部件来实现的，例如intents, system service, content provider。它们适用于不知道要通信的对象位于同一进程还是其他进程。有时候，需要定义一种更加显式的通信模型。

包括的内容如下：

- 同步和异步的远程过程调用（RPCs）
- 通过`Messenger`进行通信
- 通过`ResultReceiver`来返回数据

### Android RPC
IPC是由linux来管理的，例如：signals, pipes, message queues, semaphores, shared memory。而android中，linux的IPC由*binder framework*取代了，是进程之间的RPC。 client可以执行在server端的方法，就像在本地执行一样。因此，数据可以传递到server进程，在一个线程中处理，并且返回值到调用线程。
RPC调用本身很简单，但是其机制包括了下面的步骤：

- Method and data decomposition, also known as marshalling(方法和数据的分解)
- Transferring the marshalled information to the remote process （传递分解后的方法和数据）
- Recomposing the information in the remote process, also known as unmarshalling （在远程线程中组合分解的数据）
- Transferring return values back to the originating process （把结果返回到调用线程中）

Android平台把进程通信抽象为*binder framework* 和 *Android Interface Definition Language*.

#### Binder

*Binder*允许在不同进程的线程间传送函数和数据 - 方法调用。server进程定义了由`android.os.Binder`支持的远程接口，client进程可以通过这个远程对象访问远程接口。

传输函数和数据的远程过程调用叫做*transaction*。client进程调用`transact`方法，server进程通过调用`onTransact`接收该调用。
![IPC through Binder](http://ww4.sinaimg.cn/mw690/63293ed1jw1f5ox6l4rsrj20gp07xt9l.jpg)

client调用`transact`默认情况下被阻塞直到server端的`onTransact`返回。**传输的数据**包括**`android.os.Parcel`**对象，使得通过Binder跨进程传输极为优化。 **参数**和**返回值**当作`Parcel`对象进行传送。数据既可以是字面参数也可以是实现`android.os.Parcelable`的自定义对象。`Parcelable`是定义了数据的marshalling和unmarshalling，比`Serializable`更加有效。

`onTransact`方法使用**binder threads pool**中的thread来执行。该池只是用来处理从**其他进程发来的请求**。 最大有16个线程，这就需要调用方来保证线程安全。

IPC可以是双向的。server进程可以向client进程发送一个transaction。

> 注意：如果server进程开启一个transaction，在`onTransact`方法中发送一个请求到client 进程中，client进程不在binder thread中接收该请求，而是在等待transaction完成的线程中接收该请求。

Binder也支持异步transaction，可以通过`IBinder.FLAG_ONEWAY`来设置。这种情况下，client线程在调用`transact`之后会立即返回。而在server端的binder thread会持续的调用`onTransact`，并不能异步的返回数据的client。

#### AIDL
当进程希望向其他进程暴露函数功能的时候，就必须定义通信协议。最基本，server定义client可以调用的方法接口。最简单常用的方式是在aidl文件中描述该接口。aidl文件编译后生成的java代码会支持IPC。android应用与该生成的java代码交互，只需要知道接口的存在。

![aidl](http://ww2.sinaimg.cn/small/63293ed1jw1f5oxit88n5j20gr05n0t5.jpg)

在client应用和server应用中包含生成的java接口。
接口文件定义了两个内部类：*Proxy*和*Stub*,用来处理数据的marshalling和unmarshlling以及transaction。因此AIDL自动的生成封装binder框架的代码。

![call over aidl](http://ww1.sinaimg.cn/mw690/63293ed1jw1f5oxm5i60ij20gt0agwfo.jpg)

*Proxy*和*Stub*代表两个应用管理RPC,允许client在本地调用方法。

#### Synchronous RPC（使用AIDL）

server端是并发执行的，client端是阻塞执行的。

关于同步RPC的实例，在远程线程中返回线程名称：

1. 在.aidl文件中定义接口-通信契约，该接口描述方法定义：
```
interface ISynchronous {
    String getThreadNameFast();
    String getThreadNameSlow(long sleep);
    String getThreadNameBlocking();
    String getThreadNameUnblock();
}
```
包含*Proxy*和*Stub*内部类的java 接口使用aidl工具生成，server进程会重写*Stub*类来实现函数。

```
private final ISynchronous.Stub mBinder = new ISynchronous.Stub() {
    
    CountDownLatch mLatch = new CountDownLatch(1);
    @Override
        public String getThreadNameFast() throws RemoteException {
        return Thread.currentThread().getName();
    }
    @Override
    public String getThreadNameSlow(long sleep) throws RemoteException {
        // Simulate a slow call
        SystemClock.sleep(sleep);
        return Thread.currentThread().getName();
    }
    @Override
    public String getThreadNameBlocking() throws RemoteException {
        mLatch.await();
        return Thread.currentThread().getName();
    }
    public String getThreadNameUnblock() throws RemoteException {
        mLatch.countDown();
        return Thread.currentThread().getName();
    } 
}

```

client进程可以通过server进程的 binder对象来得到*Proxy*实现。
```
ISynchronous mISynchronous = ISynchronous.Stub.asInterface(binder);

```
*Proxy*实现了*ISynchronous*接口，在binder线程中调用远程方法。使用RPC的方式：

一些使用RPC的方式：

- *Invoking short-lived operations remotely* `mISynchronous.getThreadNameFast()`返回很快。从一个或者多个client中并行的调用会使用多个binder thread，但是由于返回很快，可以有效的重用binder threads。
- *Invoking long-lived operations remotely* `mISynchronous.getThreadNameSlow(long sleep)`client线程会被阻塞。 这样的情况多的话，会把binder thread pool 用完。
- *Invoking blocking methods* `mISynchronous.getThreadNameBlocking`，如果多个client 线程在server process中阻塞的话，binder thread pool 会用完里面的线程，导致无法开启新的线程来唤醒阻塞的进程。或造成死锁。
- *Invoking methods with shared state* 

#### Asynchronous RPC（使用AIDL）
 
与其让客户端实现异步策略，每个远程过程调用方法可以被定义为异步执行。client使用异步RPC初始化一个transaction然后返回。binder把该transaction传递给server，并且关闭server和client之间的连接。

异步RPC为了返回结果，需要使用回调函数。

异步RPC在AIDL中使用*oneway*关键字。可以被应用在interface level或者单独的方法：

*Asynchronous interface*
```
oneway interface IAsynchronousInterface {
    void method1();
    void method2();
}
```

*Asynchronous method*
```
interface IAsynchronousInterface {
    oneway void method1();
    void method2();
}
```

异步RPC最简单的形式是在方法调用中定义一个callback interface。
该回调函数是一个反向RPC，同样定义在AIDL中。例如：
```
interface IAsynchronous1 {
    oneway void getThreadNameSlow(IAsynchronousCallback callback);
}
```

server端实现远程接口的例子如下,在方法的最后，通过回调方法返回结果：
```
IAsynchronous1.Stub mIAsynchronous1 = new IAsynchronous1.Stub() {
    
    public void getThreadNameSlow(IAsynchronousCallback callback) throws RemoteException {
    
    
    callback.handleResult(xxx);
    };
    
}

```
在AIDL中定义的回调接口如下：
```
interface IAsynchronousCallback {
    void handleResult(String name);
}
```

### Message Passing Using the Binder
（上面说的是**远程方法调用(包括transact和AIDL)**，此处说的是**消息传递**）
使用*Handler*，*Looper*来传递*Message*的时候，需要线程在同一个进程中，这样会使用共享内存来存储message。如果不同的进程使用Message，此时就需要`android.os.Messenger`类来发送message到远程线程的*Handler*中。`Messenger`使用binder framework传递`Messenger`引用到client进程，也用来发送`Message`对象。`Handler`并没有被发送，相反`Messenger`作为中间者。

![messenger](http://ww2.sinaimg.cn/mw690/63293ed1jw1f5p03xcwi4j20go07wjsf.jpg)

client需要从server端得到`Messenger`引用，步骤如下：
1. 向client进程传递一个`Messenger`引用
2. client发送message到server。

#### One-Way communication
下面的例子中，Service执行在server进程并且实现了*Messenger*，Activity执行在client进程并且发送*Message*到server。
```
public class WorkerThreadServer extends Service {
    
    WorkerThread mWorkerThread;
    Messenger mWorkerMessenger;
    
    @Override
    public void onCreate() {
        super.onCreate();
        mWorkerThread.start();
    }
    
    /*
    Worker thread has prepared a looper and handler
    */
    public void onWorkerPrepared() {
        mWorkerMessenger = new Messenger(mWorkerThread.mWirkerHandler);
    }
    
    public IBinder onBind(Intent intent) {
        return mWorkerMessenger.getBinder();
    }
    
    public void onDestroy() {
        super.onDestroy();
        mWorkerThread.quit();
    }
    
    private class WorkerThread extends Thread {
        Handler mWorkerHandler;
        
        @Override
        public void run() {
            Looper.prepare();
            mWorkerHandler = new Handler(){
                public void handlerMessage(Message msg) {
                    // Implementing message processing
                }
            };
            onWorkerPrepared();
            Looper.loop();
        }
        
        public void quit() {
            mWorkerHandler.getLooper().quit();
        }
        
    }
    
}
```

client端，Activity绑定到server的Service，发送message：
```
public class MessengerOneWayActivity extends Activity {
    private boolean mBound = false;
    private Messenger mRemoteService = null;
    
    private ServiceConnection mRemoteConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className, IBinder service) {
            mRemoteService = new Messenger(service); // 通过这种方式把server的Messenger传递到client
            mBound = true;
        }
    
        public void onServiceDisconnected(ComponentName className) {
            mRemoteService = null;
            mBound = false;
        }
    };
    
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Intent intent = new Intent("com.wifill.eatservice.ACTION_BIND");
        bindService(intent, mRemoteConnection, Context.BIND_AUTO_CREATE);
    }
    
    public void onSendClick(View v) {
        if (mBound) {
            mRemoteService.send(Message.obtain(null, 2, 0, 0));
        }
    }
    
}

```

#### Two-Way Communtication

跨进程传递的*Message*在`Message.repayTo`的参数中保持了原进程中*Messenger*的引用，这是*Message*可以承载的数据类型之一。该引用可以用来在不同进程的两个线程间创建two-way通信。

下面的例子在activity和service之间创建了two-way通信。activity发送一个带有replyTo参数的message：
```
public void onSendClick(View v) {
    if (mBound) {
        try{
            Message msg = Message.obtain(null, 1, 0, 0);
            msg.replyTo = new Messenger(new Handler(){
                public void handleMessage(Message msg) {
                    Log.d(TAG, "Message sent back - msg.what = " + msg.what);
                }
            });  // 此处是关键
            
            mRemoteService.send(msg);
        } catch (RemoteException e) {
            Log.e(TAG, e.getMessage());
        }
        
    }
}

```
*Service*使用接收到的*Messenger*发送*Message*到*Activity*：
```
public void run() {
    Looper.prepare();
    mWorkerHandler = new Handler() {
        public void handleMessage(Message msg) {
            case 1:
            msg.replyTo.send(Message.obtain(null, msg.what, 0, 0)); // 此处server向client发送message
        }
    };
}

```

> 注意：*Messenger*与*Handler*结合，都属于其所属的进程，因此任务是**顺序执行**的。而AIDL由于使用**binder threads pool**可以**并行执行**。

#### Summary
进程间通信通常通过高层部件来解决。但是当需要的时候，可以使用底层的binder机制：RPC和Messenger。 如果希望提高性能（使用AIDL），使用RPC，否则Messenger更加容易实现，但是是单线程的（类似于线程间Message通信）。
本章对`transact`介绍不多。
