# 基本介绍
使用ThreadPerTaskExecutor时，每一个任务都会创建一个线程，理想情况下可以创建无限多线程。
主要为jdk21提供的虚拟线程使用的。虚拟线程比较轻量级，平台线程比较重。

# 基础使用
通过`Executors.newVirtualThreadPerTaskExecutor();`创建一个虚拟线程ThreadPerTaskExecutor，用于提交任务给虚拟线程执行。
```java
ExecutorService executor= Executors.newVirtualThreadPerTaskExecutor();
executor.submit(()->{

});
```
# 源码阅读
## ThreadPerTaskExecutor
查看ThreadPerTaskExecutor的submit方法可以知道目前Executor的流程为创建了一个ThreadBoundFuture然后执行start直接通过future对应的virtual thread进行执行task。
这里有个细节，在执行虚拟线程之前会放在set里，执行完了会删除thread。即同一时刻同一个thread只能执行一个。
java.util.concurrent.ThreadPerTaskExecutor#submit(java.util.concurrent.Callable<T>)
```java
public <T> Future<T> submit(Callable<T> task) {
    Objects.requireNonNull(task);
    ensureNotShutdown();
    var future = new ThreadBoundFuture<>(this, task);
    Thread thread = future.thread();
    start(thread);
    return future;
}
```
在start方法里会执行`JLA.start(thread, this);`来执行Thread.start方法启动线程
```java
private void start(Thread thread) {
    assert thread.getState() == Thread.State.NEW;
    // 添加对应线程到set
    threads.add(thread);

    boolean started = false;
    try {
        if (state == RUNNING) {
            // 执行线程的start
            JLA.start(thread, this);
            started = true;
        }
    } finally {
        if (!started) {
            // 从set移除对应线程
            taskComplete(thread);
        }
    }

    // throw REE if thread not started and no exception thrown
    if (!started) {
        throw new RejectedExecutionException();
    }
}
```
## JavaLangAccess
在上面的start方法里通过执行`JLA.start(thread, this);`执行线程。JLA的定义为
`private static final JavaLangAccess JLA = SharedSecrets.getJavaLangAccess();`
在JavaLangAccess的start里执行了线程的start方法。此时thread有两种可能：Thread、VirtualThread
jdk.internal.access.JavaLangAccess#start
```java
public void start(Thread thread, ThreadContainer container) {
    thread.start(container);
}
```


## ThreadBoundFuture
ThreadBoundFuture会通过ThreadPerTaskExecutor创建一个虚拟线程
```java
ThreadBoundFuture(ThreadPerTaskExecutor executor, Callable<T> task) {
    super(task);
    this.executor = executor;
    this.thread = executor.newThread(this);
}
```
ThreadPerTaskExecutor的newThread会执行VirtualThreadFactory的newThread来创建一个虚拟线程
java.util.concurrent.ThreadPerTaskExecutor#newThread
```java
private Thread newThread(Runnable task) {
    Thread thread = factory.newThread(task);
    if (thread == null)
        throw new RejectedExecutionException();
    return thread;
}
```
## VirtualThreadFactory
VirtualThreadFactory的newThread会执行newVirtualThread创建一个虚拟线程
```java
public Thread newThread(Runnable task) {
    Objects.requireNonNull(task);
    String name = nextThreadName();
    Thread thread = newVirtualThread(scheduler, name, characteristics(), task);
    UncaughtExceptionHandler uhe = uncaughtExceptionHandler();
    if (uhe != null)
        thread.uncaughtExceptionHandler(uhe);
    return thread;
}
```
在newVirtualThread会通过`ContinuationSupport.isSupported()`判断jvm是否支持虚拟线程，如果不支持就创建一个BoundVirtualThread直接当前线程执行对应task的run方法，否则创建一个VirtualThread，让挂载当前虚拟线程，执行虚拟线程，然后卸载虚拟线程
```java
static Thread newVirtualThread(Executor scheduler,
                               String name,
                               int characteristics,
                               Runnable task) {
    if (ContinuationSupport.isSupported()) {
        return new VirtualThread(scheduler, name, characteristics, task);
    } else {
        if (scheduler != null)
            throw new UnsupportedOperationException();
        return new BoundVirtualThread(name, characteristics, task);
    }
}
```
## BoundVirtualThread
BoundVirtualThread直接执行`task.run();`在当前线程执行任务
```java
public void run() {
    // run is specified to do nothing when Thread is a virtual thread
    if (Thread.currentThread() == this && !runInvoked) {
        runInvoked = true;
        task.run();
    }
}

```

## VirtualThread
VirtualThread执行挂载、执行、卸载
```java
private void run(Runnable task) {
    assert state == RUNNING;

    // first mount
    mount();
    notifyJvmtiStart();

    // emit JFR event if enabled
    if (VirtualThreadStartEvent.isTurnedOn()) {
        var event = new VirtualThreadStartEvent();
        event.javaThreadId = threadId();
        event.commit();
    }

    Object bindings = scopedValueBindings();
    try {
        runWith(bindings, task);
    } catch (Throwable exc) {
        dispatchUncaughtException(exc);
    } finally {
        try {
            // pop any remaining scopes from the stack, this may block
            StackableScope.popAll();

            // emit JFR event if enabled
            if (VirtualThreadEndEvent.isTurnedOn()) {
                var event = new VirtualThreadEndEvent();
                event.javaThreadId = threadId();
                event.commit();
            }

        } finally {
            // last unmount
            notifyJvmtiEnd();
            unmount();

            // final state
            setState(TERMINATED);
        }
    }
}
```
