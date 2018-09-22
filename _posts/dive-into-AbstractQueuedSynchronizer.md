---
title: Dive into FutureTask
date: 2018-09-19 15:49:48
tags:
---
从上一个blog上学习到了ListenableFutureTask的设计和实现（之一），其中可知，ListenableFutureTask是继承了FutureTask的。关于FutureTask，FutureTask对象可以接收其他线程的执行结果，将同步操作改为异步操作，从而提高服务的响应时间和吞吐量。FutureTask底层使用了LockSupport实现线程间的通信，那么FutureTask是如何获取其他线程的执行结果的呢？又是如何取消任务的执行的呢？

## 1.FutureTask的成员变量&静态变量
FutureTask的初始化有以下几种方式：
```java
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    /** 潜在的callable; nulled out after running */
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;
```
其中FutureTask有上面几种状态，他们的转换方式如下：
![FutureTask状态转换机](/image/futuretaskstate.webp)
其中2，3，4，6为终止状态。

其中，在调用get()方法时**阻塞**的线程会被构造成一个节点，加入到链表waiters中，waiters是链表的头结点（需要注意的是，并不是执行get()方法的线程都会被加入到改链表中）。
Waiter节点定义如下：
```java
    /**
     * Simple linked list nodes to record waiting threads in a Treiber
     * stack.  See other classes such as Phaser and SynchronousQueue
     * for more detailed explanation.
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```
存储了线程和下一个节点信息，存储线程主要是为了知道唤醒哪一个线程。
从FutureTask的成员变量中可以看出，当实例化一个FutureTask对象时，必须传入一个callable对象，即便传入的是runnable对象，最终也会转变为callable对象。

## 2.FutureTask构造函数
实例化FutureTask对象，必须传入callable对象，也可以传入runnable对象，其中第一种方法比较常见：
```java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```

## 3.执行FutureTask: run()
线程池在调用submit()方法时，会将任务封装成一个FutureTask对象。
如下是run()方法的主题逻辑：
```java
    public void run() {
        //1. 执行任务前的状态检查与设置runner线程
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            //2. 执行任务
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //4.异常结果
                    setException(ex);
                }
                if (ran)
                    //3. 正常执行结果
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                //5. 处理INTERRUPTING状态，就是等待一下
                handlePossibleCancellationInterrupt(s);
        }
    }
```
run()方法主要做了五件事情，其中主要涉及到上图中的状态转换图中的黑色内容，细节如下所示：
1. 任务运行前的状态检查
    * 任务状态必须为NEW，如果不为NEW，说明已经运行，直接返回；
    * 如果任务状态为NEW，使用CAS操作设置该该任务的执行线程给runner，如果设置不成功，说明runner的值不为null，也就是说其他线程已经在执行该任务；如果设置成功，则进行后续操作。
2. 执行任务
    * 再次验证任务状态，因为其他的线程，在1，2之间可能会取消该任务，所以再次判断；
    * 执行任务，调用callable 的 call()方法
3. 结果处理
    * 正常结果的处理代码：
    ```java
    /**
     * Sets the result of this future to the given value unless
     * this future has already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon successful completion of the computation.
     *
     * @param v the value
     */
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
    ```
    1. 使用原子操作将状态转换成COMPLETING，表示任务执行完，但是还没给outcome（成员变量）赋值；
    2. 给outcome赋值；
    3. 使用原子操作将状态修改为normal，表示正常结束了；
    4. 完成后续操作，唤醒阻塞线程（在第四步介绍）；
4. 唤醒阻塞线程
    1. 首先使用原子操作CAS移除所有的waiters，并且激活其中的thread;
    2. 依次遍历waiters中的节点，使用LockSupport.unpack()方法唤醒线程；
    3. callable置空；

*QUESTION*: 为什么要将callable置空？

源代码如下：
```java
    /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            //注意是对比q 和null，如果不为q，说明已经被其他线程改变，则失败；否则，清除并唤醒waiters中所有的thread。
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
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
                break; //跳出外层循环
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```
5. 等待interrupting状态的改变
run()方法的finally语句块中内容，因为interrupting状态不为最终状态，等待其变为最终状态。
```java
    /**
     * Ensures that any interrupt from a possible cancel(true) is only
     * delivered to a task while in run or runAndReset.
     */
    private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt

        // assert state == INTERRUPTED;

        // We want to clear any interrupt we may have received from
        // cancel(true).  However, it is permissible to use interrupts
        // as an independent mechanism for a task to communicate with
        // its caller, and there is no way to clear only the
        // cancellation interrupt.
        //
        // Thread.interrupted();
    }
```

6. 异常处理
在run方法中，如果执行 callable.call() 的过程中出现异常时，则调用setException(ex)方法来处理异常，其中处理异常的代码如下：
```java
    /**
     * Causes this future to report an {@link ExecutionException}
     * with the given throwable as its cause, unless this future has
     * already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon failure of the computation.
     *
     * @param t the cause of failure
     */
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
```
可见，先通过原子操作CAS来讲状态更新为COMPLETING，然后将结果设置为t，然后再将状态设置为最终状态EXCEPTIONAL，并执行终止执行操作。

## 3.获取FutureTask的结果：get()
FutureTask提供了两种获取任务结果的方法：
1. get()阻塞方法，会一直阻塞到任务执行完成，才会返回；
2. get(long timeout, TimeUtil unit)方法，最长阻塞timeout时间，以便任务没有完成，也会返回；

get()方法主要做了两件事情：
1. 判断任务状态，如果不为四种状态（可见状态排序），则执行awaitDone()方法；
2. 任务结束，返回任务结果。

```java
    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }

    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }
```
从源码中可以看出，主要的操作就是awaitDone()和report(s)，awaitDown()的作用是等待完成，或者在中断（或超时）时中止。
其中关于awaitDone()的源代码如下：
```java
    /**
     * Awaits completion or aborts on interrupt or timeout.
     *
     * @param timed true if use timed waits
     * @param nanos time to wait, if timed
     * @return state upon completion
     */
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }
```
从源码中可以看出，并不是所有的调用get()方法的线程，就会进入阻塞链表，只有在调用get()方法时，任务状态为new的线程才会加入阻塞链表waiters中，等待任务执行完唤醒。如果任务执行完了，表明结果会快就会准备好，只需要自旋等待即可。具体的过程分析如下：

    1. 设置超时时间（如果存在的话）；
    2. 定义等待节点，如果需要构造等待节点（是有条件的），则构造并赋值给q；
    3. queued用于标记该阻塞节点是否已经插入到了链表中，防止多次插入；
    4. 使用死循环进行操作，（不完成目的不罢休，达到目的才会break）；
    5. 判断当前线程（注意，不是runner，而是当前线程，获取任务执行结果的线程）是否被中断，如果中断，则移出该线程的阻塞节点（removeWaiter()方法，如果没有阻塞节点，该方法直接返回），并抛出中断异常
    6. 如果任务已经执行完成（处于中止状态），将等待节点的线程置为空，**返回**状态；
    7. 如果任务已经执行完，但是没有给outcome赋值，则放弃CPU资源进行等待；(无break)
    8. 如果任务还未执行（即状态为NEW)，且无等待节点的，则构造等待节点；(无break)
    9. 如果任务还没有执行，并且存在等待节点的，则将等待节点加入到等待链表waiters中，通过CAS操作采用头插入法插入节点；(无break)
    10. 如果设置了超时，则判断有无超时，如果没有设置超时，则阻塞此线程，等待任务执行完执行finishCompletion()方法唤醒；(无break)
    11. 如果设置了超时，判断有无超时，如果超时，移除等待节点，**返回**状态；
    12. 如果没有超时，调用LockSupport 的parkNanos()方法进行超时等待；(无break)

*QUESTION:* 步骤5的原理是啥？可以对应到cancel中的中断方法，如果当前线程响应中断，则移除该线程的阻塞节点，并抛出异常。

在上述方法中，存在removeWaiter()方法，整体逻辑如下：

1. 判断传入的节点是否为null，如果为null直接返回；
2. 如果不为空，首先将thread字段置null；
3. 死循环，删除该thread字段为null节点；

源代码如下：
```java
    /**
     * Tries to unlink a timed-out or interrupted wait node to avoid
     * accumulating garbage.  Internal nodes are simply unspliced
     * without CAS since it is harmless if they are traversed anyway
     * by releasers.  To avoid effects of unsplicing from already
     * removed nodes, the list is retraversed in case of an apparent
     * race.  This is slow when there are a lot of nodes, but we don't
     * expect lists to be long enough to outweigh higher-overhead
     * schemes.
     */
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }
```
其上删除逻辑就是寻找thread字段为null的节点的前节点，因为thread节点为null的就是我们要删除的节点。如果前节点存在，直接删除该节点，如果前节点不存在，说明该节点在链头，因为链表的插入操作采用的是头插法，因此，修改头结点会有线程安全问题，所以使用线程安全的CAS设置头结点的值，从而达到删除该节点的目的。如果原子操作没有删除成功，说明链表的结构发生了变化，需要重试。

get()方法中另外一个重要的方法是report(s)方法，其会根据不同的状态，返回不同的结果。源代码如下：
```java
    /**
     * Returns result or throws exception for completed task.
     *
     * @param s completed state value
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```
## 4.FutureTask的取消：cancel()

```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        //1.判断并对比状态
        if ((state != NEW) || !UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        //2.runner线程的终端操作
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```
该操作对应状态转换图中的红线部分。
1. 当 mayInterruptIfRunning 为 false 时，表示cancel()方法只能响应已经提交但是还未执行的任务，其判断任务状态必须为NEW，且CAS操作将状态从NEW 置换为CANCELED成功时，会跳转到return true语句，即cancel成功；
2. 当mayInterruptIfRunning 为true 时，前面的逻辑依旧成立，同时，cancel只能响应状态为NEW，且CAS操作将状态从NEW转换成INTERUPTING成功时，才**可能**完成取消操作；为何说可能呢？因为还需要执行线程响应该中断信号，如果执行线程不响应该信号，则该中断有没有是一样的，除非执行线程中存在影响中断的操作，否则，即便调用了interrupt()方法也不起任何作用；

关于Java中的线程中断：可参见[Java里一个线程调用了Thread.interrupt()到底意味着什么？](https://www.zhihu.com/question/41048032)
## 5.辅助工具类
使用线程安全工具Unsafe来保证线程安全的目的。
```java
    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;
    private static final long runnerOffset;
    private static final long waitersOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            stateOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("state"));
            runnerOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("runner"));
            waitersOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

```