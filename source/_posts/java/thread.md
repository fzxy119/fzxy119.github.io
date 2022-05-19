---
title: Thread
date: 2022-05-19 19:51:35
tags:
- java
categories:
- 技术
---

# JAVA Thread的状态：

```java
public enum State {
       //新建线程但是未进行start方法调用的线程
        NEW,
       //一个正在JVM中运行的线程状态，以下两种情况都是运行状态
       //1：获得cpu时间分配执行状态，2:失去cpu时间分片后的就绪状态，等待系统资源的状态，如cpu处理器
        RUNNABLE，
       //线程阻塞等待一个监视锁，获取锁后可再次执行
       // 1.如进入了synchronized 阻塞方法 
       // 2.在调用Object.wait后重入synchronized  
            //调用 wait 方法必须在同步块中，即是要先获取锁并进入同步块，这是第一次 enter
            //而调用 wait 之后则会释放该锁，并进入此锁的等待队列（wait set）中
            //当收到其它线程的 notify 或 notifyAll 通知之后，等待线程并不能立即恢复执行，因为停止的地方是在同步块内，而锁已               经释放了，所以它要重新获取锁才能再次进入（reenter）同步块，然后从上次 wait 的地方恢复执行。这是第二次 enter，               所以叫 reenter
        BLOCKED,
        //线程处于等待其他线程的一个状态
        	//不带有等待时间的Object.wait，等待别的线程唤醒，Object.notify()，Object.notifyAll(),
        	//不带有时间的Thread.join，等待合并线程的执行完成
    		//或者LockSupport.park
        WAITING，
        //线程处于有限期等待状态
        //带有等待时间的
              // Thread.sleep(long),
    		 // Object.wait(long),
              // Thread.join(long),
    		 // LockSupport.parkNanos,
   			 // LockSupport.parkUntil
        TIMED_WAITING,
        //线程处于完成终止状态
        TERMINATED;
    }
```

# JavaThread的操作

- 静态方法
  
  public static void sleep(long millis, int nanos) throws InterruptedException
  
  如果是临界区代码段，那么线程**不会释放监视器锁资源**
  
  
  
- 成员方法
  public final void join() throws InterruptedException
  public final synchronized void join(long millis, int nanos)throws InterruptedException
  public final synchronized void join(long millis) throws InterruptedException 
  
  如果是临界区代码段，那么线程**不会释放监视器锁资源**
  
  
  
  public void interrupt()
  public boolean isInterrupted() 
  
  对于正在执行的线程，终端操作只是将线程的终端标识设置为true，表示已中断，并不会做其他的操作，线程可在内部根据isInterrupted 来判断线程释放继续执行
  
  对于阻塞的线程，终端操作会直接使线程抛出InterruptedException 异常，线程根据情况判断是否继续进行线程操作
  
  
  
- LockSupport：
  LockSupport.park
  LockSupport.unpark
  LockSupport.parkNanos,
  LockSupport.parkUntil

  

- Object：
  Object.wait(long)
  Object.wait

  如果是临界区代码段，那么**线程释放监视器锁资源**

- Object.notify()，
  Object.notifyAll(),

  通知其他等待线程，锁已释放，重新竞争锁资源

# 线程创建的和使用的几种方式：

### 继承方式：

```java
class OThread extends Thread{
    @Override
    public void run() {
       log.info("线程运行");
    }
}
@Test
public void createThread(){
	OThread ot = new OThread();
	ot.start();
}
```

### 执行对象方式

Runnable对象可作为线程的执行对象来传递到线程内部并执行

```java
Runnable targ = new Runnable() {
    @Override
    public void run() {
        log.info("Runnable线程运行");
    }
};
Thread runTarget1 = new Thread(targ);
Thread runTarget2 = new Thread(targ);
runTarget1.start();
runTarget2.start();
```

### FutureTask方式:

Future接口支持我们获取异步执行的结果，并可以在执行的过程中进行cancel操作，但是无法作为线程的执行对象

Runnable可以作为线程的执行对象，但是无法获取到执行的结果，

那RunnableFuture就结合了Runnable和Future接口的对两者的功能进行了融合，既可以作为Thread的执行对象也可以获取到异步执行结果，FutureTask就是RunnableFuture的实现，来作为线程执行对象，并支持获取执行结果

```java
FutureTask fttask = new FutureTask<String>(new Callable<String>() {
     @Override
     public String call() throws Exception {
         return "FutureTask result";
     }
 });
 Thread ft = new Thread(fttask);
 ft.start();
log.info("FutureTask:{}",fttask.get());
```

### 线程池方式：

```java
//单线程，无解队列线程池LinkedBlockingQueue
//队列无线，当任务无线大时会导致内存溢出，RejectedExecutionHandler
ExecutorService executorServiceSingleThreadExecutor =  Executors.newSingleThreadExecutor();
//
//无限线程容量,SynchronousQueue队列，RejectedExecutionHandler
//对线程没有限制，有keepAliveTime60秒
ExecutorService executorServiceCachedThreadPool =  Executors.newCachedThreadPool();
//固定数量线程，无界队列线程池LinkedBlockingQueue
//队列无线，当任务无线大时会导致内存溢出，RejectedExecutionHandler
ExecutorService executorServiceFixedThreadPool =  Executors.newFixedThreadPool(3);
//计划任务线程池 DelayedWorkQueue优先级队列
ExecutorService executorServiceScheduledThreadPool =  Executors.newScheduledThreadPool(2);
//抢占式线程池
ExecutorService executorWorkStealingPool =  Executors.newWorkStealingPool();

Future<String> poolF = executorServiceSingleThreadExecutor.submit(new Callable<String>() {
    @Override
    public String call() throws Exception {
        return "executorServiceSingleThreadExecutor";
    }
});
log.info("executorServiceSingleThreadExecutor:{}", poolF.get());
```



# 锁

synchronized 内置锁

1. 无锁
2. 偏向锁,threadid
3. 轻量级自旋锁,cas,
4. 重量级锁 cxq,entrylist,waiteset,monitor

原子性操作：

1. unsafe cas volatile
2. cas 优化 LongAddr 分段锁

可见性与有序性：volatile

as-is-Serial 保障单内核指令重排序后执行结果的正确性原则，不能保证多内核以及夸cpu指令重排序之后的执行结果正确性

内存屏障     保障多核指令重排序之后程序结果正确性方法



# JMM

 内存模型

  对象的创建

  堆内存分配 指针碰撞(Serial,ParNew)，空闲列表(CMS)

  内存分配线程并发  同步锁，TLAB

  对象引用：直接指针，引用句柄

内存溢出：堆溢出 元数据区溢出，栈溢出 直接内存溢出



# 垃圾收集器及内存分配策略

哪些内存需要回收

1.    哪些区域需要回收？（堆，元数据区），（方法栈 本地方法栈 程序计数器）
2.    哪些对象需要回收？

- ​		对象已死么？引用计数法，可达性分析法
- ​        可达性分析法

​	对象已死么-GCRoot条件：

- ​	    虚拟机栈中引用的变量（栈帧中的本地变量）
- ​		元数据区中静态属性引用的对象
- ​		元数据区中常量引用的对象
- ​		本地方法栈JNI（native方法）引用的对象

   对象已死么-可达性分析法-引用：

- ​		强引用
- ​		软引用
- ​		弱引用
- ​		虚引用

​    对象已死么-是否对象已死

- ​		两次标记
- ​	    逃脱死亡的最后机会finalize

​	回收元数据区：-Xnoclassgc

垃圾回收算法：

​			标记清除算法

​			复制算法

​			标记整理算法

什么时候回收

​	枚举根节点：

​               类加载完成的时候对象内通过Oopmap数据结构记录引用信息,gc过程中直接遍历oopmap结构，解决因堆和元数据区内存比较大，导致root节点遍历时间过长。

​	 安全点：因为root节点的对象包含线程运行栈

​     安全区域：修改oopmap结构的区域

​	

# AbstractQueuedSynchronizer

模板方法

```

 private transient volatile Node head;
 private transient volatile Node tail;
 private volatile int state;

public final void acquire(int arg)
public final boolean tryAcquireNanos(int arg, long nanosTimeout)throws InterruptedException
public final void acquireInterruptibly(int arg)throws InterruptedException
public final boolean release(int arg) 

public final void acquireShared(int arg)
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)throws InterruptedException
public final void acquireSharedInterruptibly(int arg)throws InterruptedException 
public final boolean releaseShared(int arg) 

protected boolean tryAcquire(int arg)
protected boolean tryRelease(int arg)
protected int tryAcquireShared(int arg)
protected boolean tryReleaseShared(int arg)

```






