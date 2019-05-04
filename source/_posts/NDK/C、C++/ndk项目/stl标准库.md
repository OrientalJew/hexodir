---
title: C++ STL标准库
date: 2019-03-24
categories:
- C++
---
<!-- toc -->

> 相当于Java中的 util包下的工具函数，提供了各种集合和工具类

#### string

```
#include<string>

void main(){
	string s1 = "str1";
	string s2("str2");
	string s3 = s1 + s2;

	cout << s1 << endl;

	// 转C字符串
	const char* c_str = s3.c_str();

	system("pause");
}
```

#### 集合

```
#include<vector>

void main(){
	// 不需要使用动态内存分配，就可以实现动态数组
	vector<int> vi;
	vi.push_back(1);
	vi.push_back(10);

	for (int i = 0; i < vi.size(); i++){
		cout << vi[i] << endl;
	}

	system("pause");
}
```
