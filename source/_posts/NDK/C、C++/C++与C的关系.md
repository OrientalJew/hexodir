---
title: C++ 与 C的关系
date: 2019-02-17
categories:
- C++
---
<!-- toc -->

### C与C++的关系

1，C++ 是在C的基础上，对C的增强，引入了面向对象的概念(不完全面向对象)，而C只是面向过程；

2，C++ 中允许存在C代码，也就是说，C++可以与C混编，而在C中则不允许；

3，C++中支持C中的关键字和函数，同时又部分定义了自己的相同功能的函数，比如：
对于堆内存，可以通过C的malloc()和free()来管理，也可以用自己的一套new()和delete来管理；

4，C++引入了类(class)的概念，出现了对象和引用；
