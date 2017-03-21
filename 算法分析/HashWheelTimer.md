##Hash Wheel Timer

算法详细描述：[http://www.cse.wustl.edu/~cdgill/courses/cs6874/TimingWheels.ppt](http://www.cse.wustl.edu/~cdgill/courses/cs6874/TimingWheels.ppt)

时间轮定时任务处理，主要用来高效处理大量定时任务,  且任务对时间精度要求相对不高的场景  
例如场景：  
定时任务（5分钟后执行xx任务/每隔1天执行一次）  
超时控制（30s没有动作就断开连接）  
频率限制（每5s调用一次API/ 对同一个站点下页面的抓取最低要间隔5s）  

在大量定时/超时任务处理的时候，一种方式每个任务创建timer执行，一种方式，创建一个timer，扫描所有要执行的任务时间，到期执行。  
这种创建大量的timer或者大量扫描的方式，在任务量比较大的时候，cpu容易占满。

Hash Wheel Timer算法是维护类似时钟的方式，环形结构，环形由不同的桶组成，一个桶代表一段时间（越短Timer精度越高），
并用一个List保存在该格子上到期的所有任务，同时一个指针随着时间流逝一桶一桶转动，并执行对应List中所有到期的任务
（通过任务到期时间与桶的数量以及每个桶代表的时间，决定任务执行多少轮后，任务到期执行）。任务通过取模决定应该放入哪个桶上。

简单例子：
假如分8个桶，每个桶代表1秒。这时有个指针运行到2，有个10s的定时任务加入进来，  
那么它round次数=10s/8*1s=1，  
放入的桶的位置为10s-8s=2,当前位置为2，那么放入桶的序号为2+2=4，即放入第四个桶。  


netty里面有此算法实现分析下关键源码HashedWheelTimer.java：

HashedWheelTimer类参数  
```
    //利用原子变量 CAS特性，定义worker的状态修改器，实现后续worker状态字段的无锁修改，workerState必须是整数和volatile
    private static final AtomicIntegerFieldUpdater<HashedWheelTimer> WORKER_STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimer.class, "workerState");

    private final ResourceLeakTracker<HashedWheelTimer> leak;
    //执行tick处理
    private final Worker worker = new Worker();
    //工作线程
    private final Thread workerThread;
    //工作线程状态
    public static final int WORKER_STATE_INIT = 0; //初始化
    public static final int WORKER_STATE_STARTED = 1;//运行中
    public static final int WORKER_STATE_SHUTDOWN = 2;//停止
    @SuppressWarnings({ "unused", "FieldMayBeFinal", "RedundantFieldInitialization" })

    private volatile int workerState = WORKER_STATE_INIT; // 0 - init, 1 - started, 2 - shut down

    //tick时间间隔
    private final long tickDuration;
    //wheel桶数组
    private final HashedWheelBucket[] wheel;
    //求模使用，mask=wheel.len-1
    private final int mask;
    //工作线程启动锁
    private final CountDownLatch startTimeInitialized = new CountDownLatch(1);
    //超时任务放入队列，添加超时任务，先将其放入队列，执行tick过程中放入wheel桶中
    private final Queue<HashedWheelTimeout> timeouts = PlatformDependent.newMpscQueue();
    //取消超时任务队列，将取消的超时任务，放入该队列
    private final Queue<HashedWheelTimeout> cancelledTimeouts = PlatformDependent.newMpscQueue();
    //等待超时任务计数，统计等待执行的超时任务，与maxPendingTimeouts结合使用
    private final AtomicLong pendingTimeouts = new AtomicLong(0);
    //最大等待的超时任务数
    private final long maxPendingTimeouts;
     
    //工作线程，启动时间，单位纳秒
    private volatile long startTime;
 

```

HashedWheelTimer类创建实例化处理    
```
        //wheel数组桶，默认为小于ticksPerWheel的最大2的倍数，方便mask位计算，默认512
        wheel = createWheel(ticksPerWheel);
        //桶长度减1，即剩下位都为1，利用位计算得出元素桶索引
        mask = wheel.length - 1;

        // Convert tickDuration to nanos.
        //tick时间间隔，转换为纳秒
        this.tickDuration = unit.toNanos(tickDuration);

        // Prevent overflow.
        if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
            throw new IllegalArgumentException(String.format(
                    "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                    tickDuration, Long.MAX_VALUE / wheel.length));
        }
        //创建工作线程
        workerThread = threadFactory.newThread(worker);
        
        leak = leakDetection || !workerThread.isDaemon() ? leakDetector.track(this) : null;
        //最大挂起超时任务数
        this.maxPendingTimeouts = maxPendingTimeouts;
        //全局HashedWheelTimer实例创建数限制，默认64个
        if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
            WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
            reportTooManyInstances();
        }

```

HashedWheelTimer类启动  
```
 public void start() {
       //判断HashedWheelTimer状态，利用原子变量特性，实现对象字段CAS修改以及判断。
       switch (WORKER_STATE_UPDATER.get(this)) {
            case WORKER_STATE_INIT:
                //保证工作线程只启动一次
                if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                    workerThread.start();
                }
                break;
            case WORKER_STATE_STARTED:
                break;
            case WORKER_STATE_SHUTDOWN:
                throw new IllegalStateException("cannot be started once stopped");
            default:
                throw new Error("Invalid WorkerState");
        }

        // Wait until the startTime is initialized by the worker.
        //等待worker线程启动，利用startTimeInitialized的CountDownLatch特性
        while (startTime == 0) {
            try {
                startTimeInitialized.await();
            } catch (InterruptedException ignore) {
                // Ignore - it will be ready very soon.
            }
        }
 }
```

HashedWheelTimer类停止   
```
 public Set<Timeout> stop() {
        if (Thread.currentThread() == workerThread) {
            throw new IllegalStateException(
                    HashedWheelTimer.class.getSimpleName() +
                            ".stop() cannot be called from " +
                            TimerTask.class.getSimpleName());
        }

        //如果非启动状态，设置关闭状态（包括初始化和关闭状态），如果之前是初始化状态，实例数减1，返回空集合
        if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
            // workerState can be 0 or 2 at this moment - let it always be 2.
            if (WORKER_STATE_UPDATER.getAndSet(this, WORKER_STATE_SHUTDOWN) != WORKER_STATE_SHUTDOWN) {
                INSTANCE_COUNTER.decrementAndGet();
                if (leak != null) {
                    boolean closed = leak.close(this);
                    assert closed;
                }
            }

            return Collections.emptySet();
        }
       
        try {
            boolean interrupted = false;
             //启动状态下,线程存活的情况下，将工作线程中断，每次等待100ms，直到线程中断位置
            while (workerThread.isAlive()) {
                workerThread.interrupt();
                try {
                    workerThread.join(100);
                } catch (InterruptedException ignored) {
                    interrupted = true;
                }
            }
            //worker中断后，将当前线程中断
            if (interrupted) {
                Thread.currentThread().interrupt();
            }
        } finally {
            INSTANCE_COUNTER.decrementAndGet();
            if (leak != null) {
                boolean closed = leak.close(this);
                assert closed;
            }
        }
        //返回未执行的timeout
        return worker.unprocessedTimeouts();
    }
```

HashedWheelTimer类增加新timeout任务
```
    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }
        //是否限制等待执行的超时任务数，如果设置了，超过限制数，直接拒绝添加
        if (shouldLimitTimeouts()) {
            long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
            if (pendingTimeoutsCount > maxPendingTimeouts) {
                pendingTimeouts.decrementAndGet();
                throw new RejectedExecutionException("Number of pending timeouts ("
                    + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
                    + "timeouts (" + maxPendingTimeouts + ")");
            }
        }
        //HashedWheelTimer启动
        start();

        // Add the timeout to the timeout queue which will be processed on the next tick.
        // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
        //计算截止时间，截止时间=系统当前时间+截止时间-worker启动时间，这样算将tick已经执行的时间都算入,后续可以直接减去tick
        long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
        //创建超时任务
        HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
        timeouts.add(timeout);
        return timeout;
    }
```   