---
title: RxJava操作符
date: 2018-08-08
tags:
- RxJava
categories:
- 操作符
---

<!-- toc -->

#### map于flatMap的区别

map和flatMap可以说是Rx操作符中使用频率最高的操作符，它们的区别：

1、对于map，执行过程中，只有一个事件流，我们操作的是流中的内容，可以理解为操作Observable<T>
中的T，将其装换为另一种类型O,即Observable<T> ——> Observable<O>

2、对于flatMap，其作用就是把原先一个事件流转化为多个事件流，比如有一个集合，原先一个事件流，集合
中的元素是线性的，一个一个被发送，而通过flatMap，将集合中的每个元素又转化为一个新的事件流，所有
事件流同时发送结合中的每个元素，这些元素被接收时是无序的；

> 查看ObservableFlatMap源码可以发现，每一个flatMap操作符生成的Observable都开启新的、独立
的订阅流程，这些订阅流程的Observer都持有了当前ObservableFlatMap的Observer，他们发送的数据
(onNext)最终都汇聚到了ObservableFlatMap的Observer中，通过该Observer在往下一个操作符发送
(onNext);

_在将一种数据类型的Observable转换为另一种数据类型的Observable时，如果整个流程只有发送一个数据，
那么使用map进行转换即可，完全没有必要通过flatMap来转换_
