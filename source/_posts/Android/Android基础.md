---
title: Android 基础
date: 2018-07-30
tags:
- 基础
categories:
- Android
---

<!-- toc -->

#### Android 的数据存储方式


#### 在Activity启动时何时能够拿到View宽高

从Activity启动声明周期来看，即使是在onPostResume时也是拿不到宽高的！

因为和Activity中View的布局、测量、绘制相关的类ViewRootImpl在onPostResume时还没有创建，只有
在之后DecorView被addView到WindowManagerImpl中，此时才会创建ViewRootImpl实例，用来管理View
的测量绘制工作；

在ViewRootImpl的performTraversals被执行时，采取真正的测量、布局和绘制View；

performTraversals会被执行两遍，并且在开始测量、布局和绘制之前，会将post进View中的Runnable
取出执行，利用这个特点，我们可以进行双重post，在第二重post中我们就能够最新测量出来的宽高；
