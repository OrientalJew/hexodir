---
title: typedef
date: 2019-03-24
categories:
- C++
---
<!-- toc -->

  typedef的作用是，声明新的类型名来代替原有的类型名，C语言中习惯上把用typedef声明的类型
用大写字母表示。


```
typedef int p; //将p定义为int类型，定义"p i;" = “int i;”
typedef int p[10]; //将p定义为int[10]类型，定义"p i;" = “int i[10];”
typedef int* p; //将p定义为int类型，定义"p i;" = “int *i;”
typedef struct stu p; //将p定义为结构体stu类型，定义"p i;" = “struct stu i;”
typedef int p(int , int); //将p定义为int __(int ,int)类型的函数，定义"p i;" = “int i(int, int);”

typedef void* (*f_p)(); // 定义一个函数指针类型f_p
```
