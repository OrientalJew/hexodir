---
title: 标准对象
date: 2018-07-30
tags:
- Exception
categories:
- Java
---

![image](/images/exception.jpg)

java中Exception、Error都是Throwable的子类！

- Exception即异常，是程序本身可以处理的；
- Error即错误，是程序本身无法处理的，通常是系统出现故障，或者虚拟机自身发出的错误，一般不需要主动
进行处理，比如OutOfMemoryError、StackOverflowError等等；

根据异常的发生方式，可以分为检查异常和非检查异常；

- 检查异常，要求编译器必须即时处理，Java编译器会在语法中对该类型的异常进行检查，并强制要求我们
通过try-catch语句进行主动捕获，或者用throws语句在方法声明后抛出该异常，否则无法编译通过，除了
Error、RuntimeException和其子类之外的异常都是检查异常；

- 非检查异常，这类异常编译器不要求我们主动捕获或抛出，发生在程序运行期间，通常能够导致程序崩溃。
其包括了RuntimeException及其子类(nullPointException、ArrayIndexOutOfBoundException等
)和Error；
