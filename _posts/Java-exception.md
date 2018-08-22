---
title: Java exception
date: 2018-08-22 10:39:49
tags: Java
---
## 1.简介
Java 异常时定义在程序执行时，出现异常情况或问题的对象。其通常使用try....catch代码块来抛出以及捕捉。其中，需要注意的是，异常和错误在程序中是不同的；错误表示程序无法正常恢复，而异常，一般来说是可以处理的问题，例如除法中分母为0的情况。

异常代表着在程序执行过程中需要特别注意的一些异常的情况或者问题，对于这些问题，在它们出现之前进行处理是可能的。

## 2.Java异常分类
Java对异常的组织是比较简单的，其中所有的错误和异常都继承自Throwable。其中错误以及运行时异常都是未受检异常，这种问题的发生，无法让功能继续，运算无法进行，更多是因为调用者的原因导致的而或者引发了内部状态的改变导致的。那么这种问题一般不处理，直接编译通过，在运行时，让调用者调用时的程序强制停止，让调用者对代码进行修正。如我上面例子的ArrayIndexOutOfBoundsException就是这类异常。

对于已检查异常，只要是Exception和其子类都是，除了特殊子类RuntimeException体系。 这种问题一旦出现，希望在编译时就进行检测，让这种问题有对应的处理方式。这样的问题都可以针对性的处理。如ClassNotFoundException就是这类异常。

![Exception class hierarchy](/image/built_in_exception.png)

## 3.实际中处理异常的原则
1. 函数内容如果抛出已检查的异常，那么函数上必须要声明。否则必须在函数内用try-catch捕获，否则编译失败。
2. 如果调用到了声明已检查异常的函数，要么try-catch要么throws，否则编译失败。
3. 什么时候catch，什么时候throws 呢？功能内容可以解决，用catch。解决不了，用throws告诉调用者，由调用者解决 。
4. 一个功能如果抛出了多个异常，那么调用时，必须有对应多个catch进行针对性的处理。内部又几个需要检测的异常，就抛几个异常，抛出几个，就catch几个。

## 4.参考资源
1. [Java异常介绍](https://blog.csdn.net/TimHeath/article/details/53504328)
2. [Java 异常处理的误区和经验总结](https://www.ibm.com/developerworks/cn/java/j-lo-exception-misdirection/index.html)
3. [Understanding Exception Handling in Java](https://www.developer.com/java/data/understanding-exception-handling-in-java.html)