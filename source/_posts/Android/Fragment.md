---
title: Fragment
date: 2018-07-30
tags:
- 基础
categories:
- Android
---

<!-- toc -->

#### 判断Fragment可见性

* 如果Fragment是通过FragmentManager进行show和hide操作来实现显隐的，是不会调用Fragment的
的声明周期方法的，但是可以通过onHiddenChanged来进行判断；

* 如果Fragment是配合ViewPager进行显隐的，则通过Fragment的setUserVisibleHint进行判断；
> 查看FragmentPagerAdapter和FragmentStatePagerAdapter源码可以看到对setUserVisibleHint
的调用，这个方法被调用非常频繁，在进行滑动操作时，处于当前屏幕和其左右的两个Fragment都有可能被
调用到，但是只有在Fragment成为当前屏幕的Fragment时参数才会为true，需要注意判断，并且不要做太多
的耗时操作，否则会影响到切换的流畅性；

<!-- more -->
#### FragmentPagerAdapter与FragmentStatePagerAdapter的区别

* 应用场景

一般在Fragment较少的情况下，适合使用FragmentPagerAdapter；

相对的，在Fragment较多的情况下，适合使用FragmentStatePagerAdapter；

* 主要区别

FragmentPagerAdapter不会销毁Fragment实例，只是销毁Fragment的视图，在Fragment被销毁时，
只会执行到onDestoryView，Fragment中的成员数据不会被销毁；

FragmentStatePagerAdapter会完全销毁Fragment实例，在Fragment被销毁时，会执行到onDestory
方法，如果要恢复成员变量，只能通过onSaveInstanceState来进行；
