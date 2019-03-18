---
title: C++ 静态成员
date: 2019-03-03
categories:
- C++
---
<!-- toc -->

```
class Car{
public:
	char *name;
	// 静态变量
	static int num;

	// 静态函数，不能够访问类的成员变量
	static void run(){
		cout << "car running." << endl;
	}
};

// 静态变量初始化只能在类和函数外边进行
int Car::num = 4;

void main(){
	// 初始化后，静态变量才可以使用
	Car::num++;
	cout << Car::num << endl;

	// 调用静态函数
	Car::run();

	// 通过实例也是可以访问的
	Car c;
	cout << c.num << endl;
	c.run();

	system("pause");
}
```
