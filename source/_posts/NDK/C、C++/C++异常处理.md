---
title: C++ 异常处理
date: 2019-03-21
categories:
- C++
---
<!-- toc -->

#### 异常类型

```
void myThrow(int a, int b){
	if (b == 0){
		throw "除数为零";
	}
}

void myThrow2(int a, int b){
	try{
		myThrow(1, 0);
	}
	catch (char *cat){
		// 如果函数不想捕获异常，则继续往外抛
		throw cat;
	}
}

void main(){
	try{
		int age = 200;
		if (age > 100){
			throw age;
		}
	}
	// 可以抛出和捕获任意类型的异常
	catch (int result){
		cout << "age = " << result << endl;
	}
	// ...可以捕获任意类型的异常(未知异常)
	catch (...){
		cout << "未知异常" << endl;
	}

  try{
    myThrow2(1, 0);
  }
  catch (char *cat){
    cout << cat << endl;
  }

	system("pause");
}
```

**声明函数抛出异常:**

```
// throw声明函数可能会抛出的异常类型
void myThrow2(int a, int b) throw (char*, int){

}
```

#### 标准异常、自定义异常

```
#include <stdexcept>

// C++标准异常
void myThrow3(int a, int b){
	throw out_of_range("超出范围");

	throw invalid_argument("非法参数");
}
```

**自定义异常：**

```
class MyException{

};

void myThrow(int a, int b){
	if (b == 0){
		throw MyException();
	}
}

// 继承标准异常
class MyException :public exception{
public:
	MyException(char* msg) :exception(msg){}
};
```
**内部类异常**

```
class Err{
public:
	class MyException2{
	public:
		MyException2(){}
	};
};

void myThrow4(){
	// 抛出内部类异常
	// 这种写法不一定是命名空间，C++中也是有内部类的
	throw Err::MyException2();
}
```

#### 异常接收

```
void main(){
	try{
		myThrow(1, 0);
	}
	// 可以通过指针，变量和引用来接收，正确的情况应该是用引用来接收，
	// 确保在传递过程中不产生副本
	catch (MyException& e){
		cout << "MyException" << endl;
	}
	catch (out_of_range oe){
		cout << oe.what() << endl;
	}

	system("pause");
}
```
