---
title: C++ 类和结构体
date: 2019-02-20
categories:
- C++
---
<!-- toc -->

### 类

```
#define PI 3.14

// C++ 的标准输入，输出头文件
#include<iostream>

class Circle{
private:
	int r;
public:
	double getArea(){
		return r*r*PI;
	}

	void setR(int r){
		this->r = r;
	}
};

void main(){
	// C++ 中，访问成员变量和成员方法时，类是不需要进行实例化的
	// C++的类，作用更类似于结构体struct
	Circle cl;
	cl.setR(5);

	cout << "面积为：" << cl.getArea() << endl;

	system("pause");
}
```
#### 类的大小

```
class A{
public:
	int i;
	int j;
	int k;
	static int l;
};

class B{
public:
	int i;
	int j;
	int k;
	void myPrintf(){
		cout << "打印" << endl;
	}

	// 常函数，修饰的是this的内容，即是常量指针
	// 不能在函数中修改this指向的变量的内容，使函数更加安全；
	// const B* const this
	void testConst()const{
		//this->i = 10;
	}

	// B* const this
	void testConst2(){
		this->j = 10;
		// this 是指针常量，不能修改指向
		//this = (B*)0x00009;
	}
};

void main(){
	cout << sizeof(A) << endl;
	cout << sizeof(B) << endl;

	// C/C++内存分区：栈、堆、全局(静态、全局)、常量区(字符串)、程序代码区
	// 普通类属性与结构体中的属性有相同的内存布局(各属性大小的和)，
	// 静态成员存在于全局区中；
	// 函数存在于程序代码区中；
	// 类的每一个对象，其普通属性存在各自的内存中，而函数则是共享的，放在程序代码区中，
	// 当类进行调用时，通过this指针区分不同的对象调用；

	// 普通对象能调用常函数
	B b;
	b.testConst();
	b.testConst2();

	const B b1;
	// 常量对象只能调用常函数
	// 在常函数中，当前对象指向的变量内容不能被修改，能够起到防止数据成员被非法访问
	//b1.testConst2();

	system("pause");
}
```

### 结构体

```
struct Teacher{
private:
	char name[20];

public:
	int age;
	void say(){
		cout << this->age << "岁;" << endl;
	}

};

void main(){
	// C++中的结构体是不需要在前面添加struct关键字的
	// C++中，结构体与类最大的区别在于，类是可以进行继承的，而结构体是不行的；
	Teacher t1;
	t1.age = 20;
	t1.say();
	system("pause");
}
```

#### C++中的类定义步骤

1，在C++中，类和函数一般会声明在头文件中：
teacher.h
```
// C++ 中，类和函数的声明都是放在头文件中的

// 确保该头文件在多次include时，只会被引入，编译一次
#pragma once

class MyTeacher{
private:
	int age;
	char* name;
public:
	void setAge(int age);
	int getAge();
	void setName(char* name);
	char* getName();
};
```
2，实现
teacher.cpp
```
#include "teacher.h"

int MyTeacher::getAge(){
	return this->age;
}

void MyTeacher::setAge(int age){
	this->age = age;
}

void MyTeacher::setName(char* name){
	this->name = name;
}

char* MyTeacher::getName(){
	return this->name;
}
```

3，使用

```
#include "teacher.h"

void main(){
	MyTeacher t;
	t.setName = "hsh";
	t.setAge = 20;
}
```
