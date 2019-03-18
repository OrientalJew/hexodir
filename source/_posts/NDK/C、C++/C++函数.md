---
title: C++ 函数
date: 2019-02-24
categories:
- C++
---
<!-- toc -->

#### 默认参数

```
// 函数默认参数
// 一旦有一个参数设置默认值，后面的参数就必须跟着设置默认值
void testFun(int x, int y = 1, int z = 2){
}

void main(){
	testFun(1);
	system("pause");
}
```

#### 可变参数
```
// 可变参数
void fundc(int i, ...){
	// 拿到可变参数指针
	va_list args_p;

	// 开始读取时，需要指定哪个参数是可变参数前的最后一个参数
	va_start(args_p, i);
	// 拿到可变参数列表中的值
	int a = va_arg(args_p, int);
	char c = va_arg(args_p, char);
	int  b = va_arg(args_p, int);

	cout << a << "," << c << "," << b << endl;
}

void main(){
	fundc(1, 2, 'c', 12);
	system("pause");
}
```

#### 常函数

```
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
