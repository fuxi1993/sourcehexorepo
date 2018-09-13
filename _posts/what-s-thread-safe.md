---
title: What's thread-safety
date: 2018-09-05 19:53:11
tags:
---
# Thread safety
线程安全是应用在多线程代码上的一种计算机编程概念。线程安全的代码在操作共享的数据时保证了所有线程安全地按照逻辑进行执行，而不会出现意想不到的影响。有多重方法或策略来完成线程安全的数据结构。

应用程序可能在多个线程中执行同一段共享地址空间的代码，其中每一个线程可以访问到其他所有的线程中的内存。线程安全是一种允许代码执行在多线程环境中的性质，其通过[同步]( https://en.wikipedia.org/wiki/Synchronization_%28computer_science%29 )来重新建立实际控制流与程序文本之间的一些对应关系。

## 1.线程安全的级别
软件库可以提供一些线程安全的保证。比如，并发地读文本可以保证是线程安全的，但是并发地写可能就不是。使用这种库的程序是否是线程安全的取决于它是否以与这些保证一致的方式来使用库。

不同的材料对线程安全使用略有不同的术语：

* 线程安全的：实现上是保证在多个线程同时访问（执行）时竞争条件自由的。
* 条件安全的：不同的线程可以同时访问不同的对象，并且，对共享的数据是被保护起来免于竞争的。
* 线程不全的：代码不能被不同的线程同时访问（执行）。

线程安全保证通常还包括防止或者限制不同形式的死锁的风险的设计步骤，以及并发性能最大化的优化。但是，无法始终给出无死锁保证，因为死锁可能是由回调和违反独立于库本身的架构分层引起的。
## 2.方法的实现
下面讨论两种方法来达到避免条件竞争的目的。其中，第一种方法专注于避免共享状态，包括如下：

* [可重入性](https://en.wikipedia.org/wiki/Reentrancy_%28computing%29)
以这样的方式编写代码，使其可以由一个线程部分地执行，由同一个线程重新执行或者有另外一个线程同时执行，并依然正确的完成原始的操作。这需要将状态信息保存在每个执行的本地变量中，通常在堆栈上，而不是静态或全局变量或其他非本地状态。 必须通过原子操作访问所有非本地状态，并且数据结构也必须是可重入的。
* [线程本地存储](https://en.wikipedia.org/wiki/Thread-local_storage#Java)
变量已本地化，因此每个线程都有自己的私有副本。 这些变量在子例程和其他代码边界中保留其值，并且是线程安全的，因为它们对于每个线程是本地的，即使访问它们的代码可能由另一个线程同时执行。
* [不可变对象](https://en.wikipedia.org/wiki/Immutable_object)
对象的状态在构建后不可改变。这意味着只共享只读数据并获得固有的线程安全性。 然后可以以这样的方式实现可变（非常量）操作，即它们创建新对象而不是修改现有对象。 这种方法是函数式编程的特征，也可用于Java，C＃和Python中的字符串实现。

第二种类型的方法是与同步相关的，其被使用在共享状态是无法避免的情况下:

* [互斥](https://en.wikipedia.org/wiki/Mutual_exclusion)
使用确保只有一个线程可以随时读取或写入共享数据的机制来串行化对共享数据的访问。合并互斥需要经过深思熟虑，因为不当使用会导致诸如死锁，活锁和资源匮乏等副作用。
* [原子操作](https://en.wikipedia.org/wiki/Linearizability)
通过使用不能被其他线程中断的原子操作来访问共享数据。 这通常需要使用特殊的机器语言指令，这些指令可能在运行时库中可用。 由于操作是原子操作，因此无论其他线程如何访问它，共享数据始终保持有效状态。 原子操作构成了许多线程锁定机制的基础，并用于实现互斥原语。

## 3.样例
如下例子是一段Java代码，方法是线程安全的：
```java
class Counter {
    private int i = 0;
    public synchronized void inc() {
        i++;
    }
}
```
在C语言中，每一个线程都有自己的栈，但是，静态变量是不存放在栈中。所有的线程同时的访问它。如果多个线程同时运行，则一个线程可能会改变静态变量，而另一个线程可能会在一个线程更改途中检查它，造成不合意的事情发生。这种难以诊断的逻辑错误，可以在大多数时间编译和运行正常，成为竞争条件。避免这种情况的一种常见的方法是使用另一个共享变量作为“锁定”或“互斥”。

在下面的一段C语言代码中，方法是线程安全的，但是不可重入的：
```C++
# include <pthread.h>

int increment_counter ()
{
 static int counter = 0;
 static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

 // only allow one thread to increment at a time
 pthread_mutex_lock(&mutex);

 ++counter;

 // store value before any other threads increment it further
 int result = counter;

 pthread_mutex_unlock(&mutex);

 return result;
}
```
在上面，increment_counter可以由不同的线程调用而没有任何问题，因为互斥锁用于同步对共享计数器变量的所有访问。 但是如果函数在重入中断处理程序中使用并且函数内部出现第二个中断，则第二个例程将永久挂起。 由于中断服务可以禁用其他中断，整个系统可能会受到影响。

使用C++11中的无锁原子，可以将相同的函数实现为线程安全和可重入的：
```c++
# include <atomic>

int increment_counter ()
{
 static std::atomic<int> counter(0);

 // increment is guaranteed to be done atomically
 int result = ++counter;

 return result;
}
```
## 4.参考链接
* [Concurrency control](https://en.wikipedia.org/wiki/Concurrency_control)
* [Exception safety](https://en.wikipedia.org/wiki/Exception_safety)
* [Priority inversion](https://en.wikipedia.org/wiki/Priority_inversion)
* [ThreadSafe](https://en.wikipedia.org/wiki/ThreadSafe)