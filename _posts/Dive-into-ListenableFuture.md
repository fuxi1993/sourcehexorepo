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

## 3.what's more
另外ListenableFuture还有其他几种内置实现：

1. SettableFuture：不需要实现一个方法来计算返回值，而只需要返回一个固定值来做为返回值，可以通过程序设置此Future的返回值或者异常信息
2. heckedFuture： 这是一个继承自ListenableFuture接口，他提供了checkedGet()方法，此方法在Future执行发生异常时，可以抛出指定类型的异常。