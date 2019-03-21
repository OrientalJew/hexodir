---
title: C++ 模板类
date: 2019-03-21
categories:
- C++
---
<!-- toc -->

> C++模板类，相当于Java中的泛型，不过模板类的范围更加广阔，可以使用基本数据类型；

```
template<class T>
class A{
public:
	A(T t){
		this->t = t;
	}
protected:
	T t;
};

// 继承模板类
class B :public A<int>{
public:
	B(int a) :A<int>(a){

	}
};

template<class E>
class C :A<int>{
public:
	C(E e, int a) :A(a){
		this->e = e;
	}

protected:
	E e;
};

template<class T>
class D :A<T>{
public:
	D() :A(10){}

	D(T t) :A(t){
	}
};

void main(){

	A<char*> str("hihi");

	B b(10);
	C<int> c(10,10);

	D<int> d1;

	D<int> d2(10);

	system("pause");
}
```
