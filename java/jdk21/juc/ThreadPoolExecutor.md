# 总结
ThreadPoolExecutor通过使用线程池里的一个线程来执行提交的task。当任务数多的时候通过动态的创建线程到最大值来进行处理，没有任务的时候通过减少线程。
# 实战
## 自定义ThreadPoolExecutor
通过自定义ThreadPoolExecutor实现提供的hook函数来自定义功能
```java
static class LogThreadPoolExecutor extends ThreadPoolExecutor {

        public LogThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,
                                     BlockingQueue<Runnable> workQueue) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
        }

        public LogThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,
                                     BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory);
        }

        public LogThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,
                                     BlockingQueue<Runnable> workQueue, RejectedExecutionHandler handler) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, handler);
        }

        public LogThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit,
                                     BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
        }

        @Override
        protected void beforeExecute(Thread t, Runnable r) {
            System.out.println("ready for executing");
        }

        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            System.out.println("execute complete");
        }
    }
```
## 自定义ThreadFactory
自定义ThreadFactory来实现线程的命名的自定义
```java
static class NamedThreadFactory implements ThreadFactory {
        private final String namePrefix;
        private int counter = 0;

        public NamedThreadFactory(String namePrefix) {
            this.namePrefix = namePrefix;
        }

        @Override
        public Thread newThread(Runnable r) {
            return new Thread(r, namePrefix + "-" + counter++);
        }
    }
```
## 创建ThreadPoolExecutor
完整的创建ThreadPoolExecutor需要传7个参数
1. corePoolSize: 核心线程数
2. maximumPoolSize: 最大线程数
3. keepAliveTime: 线程空闲时间
4. unit: 时间单位
5. workQueue: 存放任务的队列
6. threadFactory: 创建线程的工厂
7. handler: 当任务满了后再次提交任务时的策略（拒绝、报错、当前线程执行）
```java
ThreadPoolExecutor threadPoolExecutor = new
        LogThreadPoolExecutor(1, 1, 0L,
        TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(1),new NamedThreadFactory("test"),
        new ThreadPoolExecutor.DiscardPolicy());
```
## 提交任务
通过sumbit提交不需要返回值的任务，通过`execute`提交需要返回值的任务
```java
threadPoolExecutor.submit(()->{
            System.out.println("just do it !!!");
        });
```
## 关闭线程池
通过`shutdown`关闭线程池，此时不会接受新提交的任务，会继续执行现存的任务
```java
threadPoolExecutor.shutdown();
```
# 源码分析
## submit 提交任务
通过执行ThreadPoolExecutor#submit即可提交任务。
ThreadPoolExecutor没有实现submit，而是调用的父类`AbstractExecutorService`的submit，这是一个模板方法，由子类执行execute。
### AbstractExecutorService
submit首先根据task创建了一个RunnableFuture（FutureTask），然后执行子类实现的execute
java.util.concurrent.AbstractExecutorService#submit(java.lang.Runnable)
```java
public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
}
```
newTaskFor直接创建了一个FutureTask
java.util.concurrent.AbstractExecutorService#newTaskFor(java.lang.Runnable, T)
```java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
}
```
### FutureTask
FutureTask构造函数传入了一个runnable需要执行的task，result需要返回的结果. \
新创建FutureTask时设置的状态为New，同时将runnable, result通过`Executors.callable`进行封装为RunnableAdapter \
java.util.concurrent.FutureTask#FutureTask(java.lang.Runnable, V)
```java
public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
}
private static final class RunnableAdapter<T> implements Callable<T> {
        private final Runnable task;
        private final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
        public String toString() {
            return super.toString() + "[Wrapped task = " + task + "]";
        }
}
```
### ThreadPoolExecutor
execute核心步骤：
1. **判断线程池里的work是否达到corePoolSize，如果没有达到直接创建一个work执行**
2. **如果达到了corePoolSize，将task放入workQueue队列**。放入后，如果线程池未运行则执行解决策略，如果没有正在工作的work则创建一个work
3. **如果task放入队列失败，则继续创建work直到maximunPoolSize**
4. **如果达到了maximunPoolSize,线程池将执行解决策略**
java.util.concurrent.ThreadPoolExecutor#execute
```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))  // 小于corePoolSize创建work
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) { // 大于等于corePoolSize task直接入workQueue
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false)) // 队列满了创建work
            reject(command);
}
```
## addWork 创建work
### ThreadPoolExecutor
1. 如果容器关闭了 直接返回false
2. 如果当前work数 > corePoolSize or maximumPoolSize 返回false
3. 创建work
4. 执行work
java.util.concurrent.ThreadPoolExecutor#addWorker
```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (int c = ctl.get();;) { // 通过ctl判断当前TheadPoolExecutor的状态
            // Check if queue empty only if necessary.
            if (runStateAtLeast(c, SHUTDOWN)  // 判断容器是否关闭
                && (runStateAtLeast(c, STOP)
                    || firstTask != null
                    || workQueue.isEmpty()))
                return false;

            for (;;) {
                if (workerCountOf(c) // 判断当前work的数量
                    >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                    return false;
                if (compareAndIncrementWorkerCount(c)) // 增加work数量
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateAtLeast(c, SHUTDOWN))
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask); // 创建work
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();

                    if (isRunning(c) ||
                        (runStateLessThan(c, STOP) && firstTask == null)) { // 如果容器正在运行或者正在关闭同时新加了任务就添加work
                        if (t.getState() != Thread.State.NEW)
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        workerAdded = true;
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) { 
                    container.start(t); // 执行work
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
### Worker
Work继承了AbstractQueuedSynchronizer实现了自己的同步策略，同时实现了Runnable接口，本身可以作为一个task执行
```java
   private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable{
}
```
Worker构造函数里设置了task、thread \
同时Work重新实现的run方法执行了ThreadPoolExecutor的runWorker方法
```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
public void run() {
    runWorker(this);
}
```
runWorker方法为核心的执行任务流程
1. 如果task为空通过getTask从workQueue获取task执行
2. worker加锁（继承了AQS）
3. 如果当前ThreadPoolExecutor就停止当前运行的线程
4. 执行beforeExecute 任务执行前的hook
5. 执行task(FutureTask)
6. 执行afterExecute 任务执行后的hook，注意如果第二个参数不为空表示有异常
7. 执行processWorkerExit 维持核心线程数
```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    try {
                        task.run();
                        afterExecute(task, null);
                    } catch (Throwable ex) {
                        afterExecute(task, ex);
                        throw ex;
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```
## core work 保存
processWorkerExit用于work执行完退出的处理以及核心work的维持
### processWorkerExit
processWorkerExit核心流程
1. 记录当前完成的任务数，同时移除当前work
2. 如果容器还在运行，同时当前work Count > corePoolSize 则 结束当前work
3. 否则就重新创建一个work，以维持corePoolSize个work
java.util.concurrent.ThreadPoolExecutor#processWorkerExit
```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```
### getTask
getTask核心逻辑：
1. 如果ThreadPoolExecutor停止了就结束当前work
2. 如果work > corePoolSize 同时work在keepAliveTime时间内未获取到task，则结束当前work
3. 否则 work <= corePoolSize 会执行 `workQueue.take()`阻塞当前核心work直到有新任务。这一步就是维持核心work不结束的真正原因
java.util.concurrent.ThreadPoolExecutor#getTask
```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();

            // Check if queue empty only if necessary.
            if (runStateAtLeast(c, SHUTDOWN) // 然后ThreadPoolExecutor关闭了则结束当前work
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
 
            if ((wc > maximumPoolSize || (timed && timedOut)) // 如果 wc > corePoolSize 同时超时了就结束当前work
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c)) // 减少workCount
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : // 尝试阻塞获取work，如果在指定的时间（空闲时间）未获取到就结束work
                    workQueue.take(); // 维持核心work不结束的真正原因
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```
ThreadPoolExecutor将work的空闲时间转为->阻塞获取task的时间. \
注意work是从workQueue获取task的，每创建一个task都放入了workQueue。Work实现了Runnable也放入了workQueue。workQueue类型为`BlockingQueue<Runnable>`

## FutureTask
FutureTask封装了真正执行的task、thread。是work真正执行的对象。最终执行task，同时唤醒阻塞获取结果的thread
### run
FutureTask的run方法核心逻辑
1. 执行task
2. 设置执行的结果（set）
3. 缓存阻塞等待结果的线程（Callable）（set）
```java
    public void run() {
        if (state != NEW ||
            !RUNNER.compareAndSet(this, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) { // 如果当前状态为New
                V result;
                boolean ran;
                try {
                    result = c.call(); // 执行task
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result); // 设置结果
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
### set
FutureTask的set方法：
1. 设置当前FutureTask的状态为完成中
2. 将执行的结果复制到outcome。后续可以通过`get`方法获取
3. 设置完成状态
4. 完成后的处理
```java
    protected void set(V v) {
        if (STATE.compareAndSet(this, NEW, COMPLETING)) {  // 设置当前FutureTask的状态为完成中
            outcome = v; // 将执行的结果复制到outcome。后续可以通过`get`方法获取
            STATE.setRelease(this, NORMAL); // final state // 设置完成状态
            finishCompletion(); // 完成后的处理
        }
    }
```
### finishCompletion
finishCompletion为执行完成后的处理，主要为唤醒调用`get`阻塞的线程
```java
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (WAITERS.weakCompareAndSet(this, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```
### get
get方法用于阻塞获取结果
1. 如果task未执行完，调用awaitDone阻塞当前线程
2. 否则调用report获取结果
```java
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```
### awaitDone
awaitDone的主要流程：
1. 创建一个WaitNode
2. 加入FutureTask的阻塞队列waiters
3. `LockSupport.park`挂起当前线程
```java
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        long startTime = 0L;    // Special value 0L means not yet parked
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            int s = state;
           //...
            else if (q == null) {
                if (timed && nanos <= 0L)
                    return s;
                q = new WaitNode(); // 创建一个WaitNode
            }
            else if (!queued)
                queued = WAITERS.weakCompareAndSet(this, q.next = waiters, q); // 加入FutureTask的阻塞队列waiters
            else if (timed) {
                final long parkNanos;
                if (startTime == 0L) { // first time
                    startTime = System.nanoTime();
                    if (startTime == 0L)
                        startTime = 1L;
                    parkNanos = nanos;
                } else {
                    long elapsed = System.nanoTime() - startTime;
                    if (elapsed >= nanos) {
                        removeWaiter(q);
                        return state;
                    }
                    parkNanos = nanos - elapsed;
                }
                // nanoTime may be slow; recheck before parking
                if (state < COMPLETING)
                    LockSupport.parkNanos(this, parkNanos);
            }
            else
                LockSupport.park(this); // 挂起当前线程
        }
    }
```
### report
report用于获取结果。如果任务取消了则会报错。如果是其他的状态也会报错
```java
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```

## ctl->ThreadPoolExecutor状态
ctl维持了整个ThreadPoolExecutor的状态，包括：当前的状态、当前的workCount。使用前三位表示状态，使用后29位表示workCounr。所以ThreadPoolExecutor理论上最大workCount=(2^29)-1
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```
### 获取workCount
直接按位与COUNT_MASK
```java
private static int workerCountOf(int c)  { return c & COUNT_MASK; }
```
### 获取ThreadPoolExecutor状态
```java
private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
```
### 判断isRunning
```java
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
```
### 增加workCount
通过cas实现
```java
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }
```
### 减少workCount
```java
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }
```

