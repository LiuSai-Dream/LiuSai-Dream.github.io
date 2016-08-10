---
layout: post
title: Efficient Andriod Threading - Chapter 9
category: Android
---

## 9. Control over Thread Execution Through the Executor Framework

Executor framework包含的**不只是线程池**还包括**Future**等。本章包括：

- 设置worker 线程池和队列控制等待执行任务的数量
- 检测导致线程异常终止的原因
- 等待线程执行完毕，得到结果
- 批量执行线程，并以固定的顺序得到他们的结果
- 在合适的时间启动后台进程

### Executor
*Executor framework*最基本的部件是`Executor`接口。它的主要目标是为了分开任务的创建（例如*Runnable*）和执行。该接口只有一个方法：

```
public interface Executor {
    void execute(Runnable command);
}
```

尽管简单，确实框架的基础，因为提交任务和任务执行的分离，它比`Thread`接口用的更加频繁。

其他的执行行为可以通过控制来实现：

- 任务入队
- 任务执行顺序
- 任务执行类型（顺序或者并行）

一个顺序执行的例子：

```
private static class SerialExecutor implements Executor {
    
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable> ();
    Runnable mActive;
    
    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        } );
        if (mActive  == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if (mActive == mTasks.poll() != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }

}

```

其执行的顺序：

- *Task queueing* 
- *Task execution order* 
- *Task execution type* 任务顺序的执行，但是并不一定在同一个线程上。当一个任务停止（r.run()）执行的时候，`scheduleNext()`会被调用。

可以看到相比于`Thread`，`Executor`的优点在于能够对任务进行调度，存储等操作。
现有的最有用的executor实现是线程池。

### Thread Pools

线程池是任务队列和worker thread之间组成的生产者-消费者设置。

线程池的优势：

- worker thread可以保持active来等待新的任务。线程并不需要每次都创建和销毁。
- 具有最大数量的线程
- 线程的生命周期由线程池来控制


#### Prefefing Thread Pools

该框架包含了预定义的线程池，从`Executors`类中创建：

- *Fixed size* 保持有一定数量的线程。结束的线程被新开启的线程所代替。使用`Excutors.newFixedThreadPool(n)`来创建。此种类型的线程池具有**无限大小**的任务队列
- *Dynamic szie* 动态大小-缓存的线程池在有任务要处理的时候创建一个新的线程。空闲的线程等待60秒然后才终止。此种类型的线程池里的线程动态增长和缩小。使用`Executors.newCachedThreadPool()`
- *Single-thread executor* 只有一个线程。使用`Executor.newSingleThreadExecutor()`来创建。

> 注意：`Executor.newSingleExecutor()`和`Executor.newFixedThreadPool(1)`都只有一个线程来执行。 然而对于`fixedthreadpool`后续可以改变其线程的数量。

```
ExecutorService executor = Executors.newFixedThreadPool(1);
((ThreadPoolExecutor) executor).setCorePoolSize(4);
```

#### Custom Thread Pools

预定义的线程池基于`ThreadPoolExecutor`。可以对线程池进行极大的配置和扩展。

##### ThreadPoolExecutor configuration

线程池的行为基于线程的性质和任务队列，可以用来控制池。由`ThreadPoolExecutor`使用的性质可以用来定义线程的创建和终止以及任务的入队。在构造函数中设置：

```
ThreadPoolExecutor executor = new ThreadPoolExecutor(
int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue
);

```

- *Core pool size* : 线程池中的最少线程数量。实际上，线程池从0个线程开始，但是一旦达到了*Core pool size*，那么线程的数量就不会低于该大小。如果线程池的数量**低于该大小**，当有新的任务，**即使线程池中有空闲线程也会创建新的线程**。
- *Maximum pool size*： 最大可以并行执行的最大线程数量，达到该数量新建的任务就会插入到队列中。
- *Maximum idle time(keep-alive time)* ： 空闲进程可以保持存活的时间，如果设置了存活时间，系统可以回收*noncore pool*线程。
- *Task queue type* :**BlockingQueue**的实现。

可以看到*Core pool size*控制了线程的数量，但是*Maximum pool size*控制了这些线程中可以并发执行（可以调度？）的数量。

#### Designing a Thread Pool

创建线程池的目标是以硬件允许的最高速度来处理任务，而不用消费多于的内存。

##### *Sizing*
最重要的是定义线程池的最大数量。如果最大线程过低，那么线程队列中就会积压过多的未处理线程。相反，过多的线程也会导致性能的下降，调度器会把更多的精力放在管理线程上。
最好基于硬件来决定线程池的大小。主要是CPU的数量。android可以通过`int N = Runtime.getRuntime().availableProcessors()`来获取cpu的数量。N是可以并行执行的最大数量。适用于独立的非阻塞的任务。然而，所有的线程都有可能被硬件以不同的原因终止。所以需要额外的线程来充分利用CPU。
当任务与线程池中的其他任务有交互，该线程就有可能等待其他的线程为造成阻塞， 那么就不能根据硬件来决定了。在这种情况下，**设置过少的线程可能会导致deadlock**。

#### Dynamics

**除非定义一个固定大小的线程池**，线程的数量可以在池的生命周期中变化。这些变化与*core threads*和*keep-alive*有关。

线程池定义了*core thread*，保持存活。默认情况下，线程池中的数量在*core thread*和*maximum thread*之间浮动。因此如果*core thread*和*maximum thread*之间的数值差不多，线程池就会是静态的。
如果设置空闲时间为0，空闲的线程会与线程池存活时间一样长。

#### Bounded or unbounded task queue
线程池可以与*bounded*或者*unbounded*队列一起使用。*unbounded*队列允许内存被消耗完。如果需要*bounded*队列，需要设置*size*和*saturation*。
使用`LinkedBlockingQueue`(bounded / unbounded),`PriorityBlockingQueue`,`ArrayBlockingQueue`(bounded)。

#### Thread configuration

`ThreadPoolExecutor`不止定义了线程的数量，线程池的创建和终止，也可以定义**每个线程的性质**。

一种通用的策略是降低线程的优先级，使得UI线程能够得到更快的响应。

通过`ThreadFactory`接口来实现线程的配置。

```
class LowPriorityThreadFactory implements ThreadFactory {

    private static int count = 1;
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r);
        t.setName("LowPrio " + count++);
        t.setPriority(4);
        t.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler()
        {
            @Override
            public void uncaughtException(Thread t, Throwable e)
            {
            Log.d(TAG, "Thread = " + t.getName() + ", error = " +
         e.getMessage());
         }
        });
        return t;
    
    }

}

Executors.newFixedThreadPool(10, new LowPriorityThreadFactory());

```

由于线程池有多个线程，会与UI线程竞争执行时间，那么把线程设置为低的优先级会更好。

#### Extending ThreadPoolExecutor

可以通过扩展`ThreadPoolExecutor`来跟踪*executor*的执行或者其任务，应用可以定义下面的方法来做些工作：

- `void beforeExecute(Thread r, Runnable r)` 在**执行一个线程之前**，由运行时库执行
- `void afterExecute(Runnable r, Throwable t)` **在线程终止之后**，由运行时库执行。不管有没有异常发生
- `void terminated()` 在**线程池关闭后**，由运行时库执行，此时没有更多的任务执行或者等待执行

示例：

```
public class TaskTrackingThreadPool extends ThreadPoolExecutor {
    
    private AtomicInteger mTaskCount = new AtomicInteger(0);
    
    public TaskTrackingThreadPool() {
        super(3,3,0l, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>());
    }
    
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        mTaskCount.getAndIncrement();
    }


    protected void afterExecute(Runnable r, Throwable t) {
    super.afterExecute(r, t);
    mTaskCount.getAndDecrement();
    }
public int getNbrOfTasks() {
    return mTaskCount.get();
    }

}

```

#### Lifecycle

线程池的生命周期从其创建到所有线程的终止。通过`ExecutorService`接口来管理，该接口扩展`Executor`类并由`ThreadPoolExecutor`实现。

![Thread pool lifecycle](http://ww4.sinaimg.cn/mw690/63293ed1jw1f5sk7g1z2ij20h10403yp.jpg)

- *Running* 线程池创建的初始状态，接收新的*task*，并在*thread*中执行
- *Shutdown* 调用`ExecutorService.shutdown` 把已有的线程处理完，不再接收新的线程
- *Stop* 调用`ExecutorService.shutdownNow`，当前线程被*interrupted*，队列中的任务被清空
- *Tidying* 内部清理
- *Terminated* 最终的状态，没有线程和任务。`ExecutorService.awaitTermination`停止阻塞，调用了`ThreadPoolExecutor.terminated`。

生命周期状态无法反转，一旦线程池为Running 状态，就被初始化到终止的状态，并且之后无法重复使用。唯一可以控制的状态是选择*Shutdown*或者*Stop*状态。
因此必须设置*cancellation policy*来终止线程的执行和内存的回收。

#### Shutting Down the Thread Pool

`void shutdown()`或者`List<Runnable> shutdownNow()`来显示的终止线程池。

| Figure reference | shutdown() | shutdownNow() |
| -------- | ---------- | :------------ |
| 1. Newly added tasks | New tasks are rejected | New tasks are rejected |
| 2. Tasks pending in the queue | Pending tasks will be executed | Pending tasks are not executed, but returned instead to a List<Runnable> so that they potentially can be executed on other therads |
| 3. Tasks being processed | Processing continues | Threads are interrupted |

![Executor shutdown](http://ww4.sinaimg.cn/mw690/63293ed1jw1f5skmvfu5wj20gp03m3yo.jpg)

当线程池**没有被手动关闭**，**线程池内没有线程**并且**不再被应用引用**的时候会自动关闭。
然而如果没有设置*keep-alive*时间的话，线程会保持空闲状态。**自动关闭只对所有线程都设置了kepp-alive时间的线程池有效**。

总结：关闭线程池的二个方法：1. 手动的关闭线程池；2. 设置了*keep-alive*参数（用来终止线程的）并且线程池对象没有被引用的时候；
当没有设置*keep-alive*参数的时候无法自动的关闭线程池。

> 注意：如果线程池初始化为*shutdown为，无法被任务重用。就必须为后续的任务开启一个新的线程池，或者自己执行`shutdownNow()`返回的任务。

#### Thread Pool Uses Cases and Pitfalls

一些不寻常的使用线程池方法，以及自定义线程池的陷阱。

###### Favoring thread creation over queuing
默认情况下，*alive time*不适用于*core theads*。但是`allowCoreThreadTimeOut(true)`可以达到目的。

```
int N = Runtime.getRuntime().availableProcessors();

ThreadPoolExecutor executor = new ThreadPoolExecutor(N * 2, N * 2, 60L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>);
executor.allowCoreThreadTimeOut(true);

```

###### Handling preloaded task queues

线程池**刚创建**的时候**没有线程存在**。当任务**提交到线程池**的时候才会进行线程的创建。没有任务**提交**就没有线程的创建，尽管队列中有待处理的任务。
可以通过使用`prestartAllCoreThreads()`或者`prestartCoreThread()`在`ThreadPoolExecutor`实例中预先启动*core thread*来执行预加载的任务队列。

```
BlockingQueue<Runnable> preloadedQueue = new LinkedBlockingQueue<Runnable>();

final String[] alphabet = {"Alpha","Beta", "Gamma","Delta","Epsilon","Zeta"};

for(int i = 0; i < alphabet.length; i++){
    final int j = i;
    preloadedQueue.add(new Runnable() {
    @Override
     public void run() {
        // Do long running operation
        // that uses "alphabet" variable.
       }
    });
}
ThreadPoolExecutor executor = new ThreadPoolExecutor(
5, 10, 1, TimeUnit.SECONDS, preloadedQueue);

executor.prestartAllCoreThreads();
```

###### The danger of zero core threads


### Task Management

运行任务的执行环境也是可以管理的。

#### Task Representation

任务本身应该独立于如何执行。也就是任务和执行环境是不同的个体。

之前看到的任务是*Runnable*形式，还有更加强大的*Callable*形式。可以通过*Future*接口来管理任务和取得结果。

*callable*定义了*call*方法，可以返回值并且能够抛出异常：

```
public interface Callable<V> {
    public <V> call() throws Exception;
}
```

*Callback*不能被*thread*实例执行，其执行环境需要基于**ExecutorService**的实现-线程池来处理任务。然后就可以通过`Future`接口来观察和控制。
由`Future`提供的方法包括：

```
boolean cancel(boolean mayInterruptIfRunnint)
V get()
V get(long timeout, TimeUnit unit) //如果没有在特定的时间内结束，会返回null值
boolean isCancelled()
boolean isDone()
```

如果任务没有成功， 可能会抛出`ExecutionException`异常。

已经提交的任务可以被取消，如果该任务在队列中，会被移除。如果当前正在执行，`cancel(false)`不会受到影响，但是`cancel(true)`会中断当前的任务。

`isCancelled()`只是查询是不是有人希望该任务取消，如果其返回`true`也不代表该任务没有执行过。
`isDone()`查询是否真正的被取消了或者成功结束了或者抛出一个异常。

#### Submitting Tasks

在任务提交到线程池之前，默认是一个没有线程的空队列。**线程池的状态**以及**队列**决定了线程池如何对新的*task*反应：

- 如果*core pool size*没有达到，立即创建新的
- 如果*core pool size*已经达到，并且可以入队，把任务入队
- 如果*core pool size*已经达到，队列是满的，任务会被rejected

可以单个以及批量提交，批量提交：`invokeAll`和`invokeAny`。

##### Individual submission
最基本的向线程池中增加任务的方式：

```
ExecutorService executor = Executor.newSingleThreadExecutor();

executor.execute(new Runnable(){
    public void run() {
        doLongRunningOperation();
    }
});

```

*Executor*接口只能处理*Runnable*任务，而*ExecutorService*能够处理*Runnable*和*Callable*。每个提交的任务由*Future*来管理和观察。但是只有*Callable*能够用来获得结果：

*Callable*:

```
ExecutorService executor = Executor.newSingleThreadExecutor();

Future<Object> future = executor.submit(new Callable<Object>() {
    public Object call() throws Execution{
        Object object = doLongRunningOperation();
        
        return object;
    }
});

// Blocking call - Returns 'object' from the callable
Object result = future.get();
```

*Runnable without result*

```
ExecutorService executor = Executors.newSingleThreadExecutor();

Future<?> future = executor.submit(new Runnable() {
    public void run() {
        doLongRunningOperation();
    }
});

// Blocking call - Always return null
Object result = future.get();

```

##### invokeAll

`ExecutorService.InvokeAll`并行执行多个独立的任务，并且通过阻塞调用线程让应用等待所有的任务完成或者超时：

```
List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)

```

```
public class InvokeActivity extends Activity {
    public void onButtonClick(View v) {
        SimpleExecutor simpleExecutor = new SimpleExecutor();
        
        simpleExecutor.execute(new Runnable(){
            
            public void run() {
                List<Callable<String> > tasks = new ArrayList<Callable<String> >();
                tasks.add(new Callable<String>() {
                    public String call() throws Exception {
                        return getFirstDataFromNetwork();
                    }
                });

tasks.add(new Callable<String>() {
    public String call() throws Exception {
        return getSecondDataFromNetwork();
    }
});
                
            ExecutorService executor = Executors.newFixedThreadPool(2);
            try {
                List<Future<String>> futures = executor.invokeAll(tasks);
                String mashedData = mashResult(futures);
            } catch (Exception e)
            
            
            }
            executor.shutdown();
        });
    
    
    }


}


```


##### invokeAny
`ExecutorService.invokeAny`执行任务集合，返回第一个完成的任务，抛弃掉剩下的任务。其适用的场景：在不同的集合中查找数据，当找到的时候立即返回。

```
<T> invokeAny(Collection <? extends Callable<T>> tasks)

<T> invokeAny(Collection <? extends Callback<T>> tasks, long timeout, TimeUnit unit)

```

其阻塞直到有任务返回结果，然后通过`future.cancel(true)`取消余下的任务。如果余下的任务没有设置终端策略，任务会继续执行完毕（因此此种情况下需要实现取消策略）但不会返回结果。

#### Rejecting Tasks

任务的添加会因为worker线程的数量和队列满，或者*executor*初始化了*shutdown*而失败。应用可以通过为线程池提供一个`RejectedExecutionHandler`接口的实现来自定义*rejection*策略。

```
void rejectedExecution(Runnable r, ThreadPoolExecutor executor)
```

平台本身提供了四种方法来处理这种情况，作为`ThreadPoolExecutor`的内部类：

- AbortPolicy
- CallerRunsPolicy
- DiscardOldestPolicy
- DiscardPolicy


##### ExecutorCompletionService

线程池可以管理任务队列和工作线程，但是无法管理完成任务的结果。通过`ExecutorCompletionService`（基于`BlockingQueue`）来完成。当任务结束后，`Future`对象会入队，由消费者线程处理。

![ExecutorCompletionService](http://ww2.sinaimg.cn/mw690/63293ed1jw1f5smgv2do3j20gp02l74d.jpg)

在`Activity`中显示图片是其一个应用。