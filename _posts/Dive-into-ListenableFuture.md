---
title: Dive into ListenableFuture
date: 2018-09-12 19:49:01
tags:
---

## 1.为什么要可监听的Future
从java1.5开始，提供了Callback和Future，通过他们可以在任务执行完毕之后得到任务执行的结果。Future可以对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过get方法获取执行结果，该方法会阻塞直到任务返回结果。
在java.util.concurrent包中，它是一个接口：
```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
其中，在调用get()方法获取结果时，其会一直阻塞等待，直至计算完成，取得结果（当然，在这个线程未被取消，未被中断，未出现异常前提下)。或者还有另外一种方法就是不停调用isDone()来查看任务是否完成，一旦完成就调用get()方法获取结果。这样做，代码结构复杂，且效率低下，所以使用ListenableFuture可以帮助检测Future是否完成，如果完成了就自动调用回调函数，这样可以减少并发程序的复杂度。

## 2.ListenableFuture
ListenableFuture是一个接口，其继承了Future的接口，达到可监听的目的：
```java
import java.util.concurrent.Executor;
import java.util.concurrent.Future;

public interface ListenableFuture<V> extends Future<V> {
    void addListener(Runnable var1, Executor var2);
}
```
其中val1是用来执行回调的Runnable, val2是用来执行val1。

### 2.1初始化
可以通过MoreExecutors类的静态方法初始化一个ListeningExecutorService的方法，然后使用此实例的submit方法即可初始化一个ListenableFuture对象，如listeningDecorator:
```java
    public static ListeningExecutorService listeningDecorator(ExecutorService delegate) {
        return (ListeningExecutorService)(delegate instanceof ListeningExecutorService ? (ListeningExecutorService)delegate : (delegate instanceof ScheduledExecutorService ? new MoreExecutors.ScheduledListeningDecorator((ScheduledExecutorService)delegate) : new MoreExecutors.ListeningDecorator(delegate)));
    }
```
其中ListenableFutre的接口如下：
```java
public interface ListeningExecutorService extends ExecutorService {
    <T> ListenableFuture<T> submit(Callable<T> var1);
    ListenableFuture<?> submit(Runnable var1);
    <T> ListenableFuture<T> submit(Runnable var1, T var2);
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> var1) throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> var1, long var2, TimeUnit var4) throws InterruptedException;
}
```

### 2.2使用ListenableFuture实例
有了ListenableFuture实例，有两种方法可以执行此Future并执行Future完成之后的回调函数。
* 方法一：通过ListenableFuture的addListener方法
* 方法二：通过Futures的静态方法addCallback给ListenableFuture添加回调函数
```java
public static <V> void addCallback(ListenableFuture<V> future, FutureCallback<? super V> callback) {
        addCallback(future, callback, MoreExecutors.sameThreadExecutor());
}

public static <V> void addCallback(final ListenableFuture<V> future,
      final FutureCallback<? super V> callback, Executor executor) {
    Preconditions.checkNotNull(callback);
    Runnable callbackListener = new Runnable() {
      @Override
      public void run() {
        final V value;
        try {
          // TODO(user): (Before Guava release), validate that this
          // is the thing for IE.
          value = getUninterruptibly(future);
        } catch (ExecutionException e) {
          callback.onFailure(e.getCause());
          return;
        } catch (RuntimeException e) {
          callback.onFailure(e);
          return;
        } catch (Error e) {
          callback.onFailure(e);
          return;
        }
        callback.onSuccess(value);
      }
    };
    future.addListener(callbackListener, executor);
  }
```

其中可以看一下getUninteruptibly方法的实现：
```java
    public static <V> V getUninterruptibly(Future<V> future) throws ExecutionException {
        boolean interrupted = false;

        try {
            while(true) {
                try {
                    Object var2 = future.get();
                    return var2;
                } catch (InterruptedException var6) {
                    interrupted = true;
                }
            }
        } finally {
            if (interrupted) {
                Thread.currentThread().interrupt();
            }

        }
    }
```
推荐使用第二种方法，因为第二种方法可以直接得到Future的返回值，或者处理错误情况。本质上第二种方法是通过调动第一种方法实现的，做了进一步的封装。

## 3.ListenableFuture的实现之一：ListenableFutureTask
ListenableFutureTask的实现逻辑如下，细节可见参考文档1
![ListenableFutureTask类图](/image/ListenableFutureTask.png)

## 4.总结&思考

其实我在读到使用推荐的方法二来进行添加回调的源代码时，就有一个疑问，即addCallback()方法只是在里面定义了一个Runnable来封装了回调的几个方法（成功，失败等），但是并没有有线程来执行这个Runnable，后面我读到 ListenableFutureTask 时，发现这个方法中的addListener方法，以及done()方法使用代理ExecutorList来通过我们生成的线程池运行了Runnable，就完整明白了其中的逻辑。

通过这次的ListenableFuture的分析学习，更加深刻的理解了Java中接口的意义，其实际上是约束了一组行为规范，但是并不具体实现，其可以认为是一个框架制定者，但是并不限制具体的实现。但是我们在实际的应用程序中，使用和接触的是实际的类对象实例，这个类如果实现了接口，就一定会实现这组行为规范，进行落地，让它真正的work起来。有时候在阅读代码或者编写代码时，就会容易犯我前面的糊涂，即担心Futures.addCallback()中的Runnable没人做，实际上我们写的时候是面向接口编程，但是在用的时候，是使用的实际的类，这些类就会完成实际的工作。这种思维无论是在阅读源码还是编写Java代码时，都是非常有用的。

## 5.参考文档
1. [《线程池系列六》-Guava ListenableFutureTask](https://www.jianshu.com/p/a4b4159163fd)