# 总结
ThreadPoolExecutor通过使用线程池里的一个线程来执行提交的task
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
