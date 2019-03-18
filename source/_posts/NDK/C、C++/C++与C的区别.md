---
title: C++ 与C 的区别
date: 2019-02-20

categories:
- C++
---
<!-- toc -->

### bool

```
// C++ 的标准输入，输出头文件
#include<iostream>

void main(){
	// C++ 比C多了一种基本数据类型，bool
	// true和false对应C中的1和0
	//bool isSingle = true;
	//bool isSingle = 3;
	//bool isSingle = -3;
	// 0表示false，非0表示true
	bool isSingle = 0;
	if (isSingle){
		cout << "single" << endl;
	}
	else{
		cout << "not single" << endl;
	}

	system("pause");
}
```

### 三元表达式

```
// C++ 的标准输入，输出头文件
#include<iostream>

void main(){

	// 在C++中，三元运算符?:并不是表示普通的逻辑运算语句(语句(statement)：不返回值的代码块，或可执行代码行)，而是可以是表达式
	// (表达式：允许有返回值)，它的返回值是逻辑运算的结果
	// 在C中，三元运算符仍然只是一个语句！
	int a = 1, b = 3;
	// 对逻辑运算结果进行赋值
	// a = 1,b= 10
	(a > b ? a : b) = 10;
	cout << "a = " << a << ",b= " << b << endl;

	system("pause");
}
```
