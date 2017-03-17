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

```
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