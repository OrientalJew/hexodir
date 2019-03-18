---
title: dll
date: 2018-07-29
tags:
- dll
categories:
- C
---
<!-- toc -->

#### visual studio 生成dll
> dll是C中的动态库，在程序执行时，被动态加载进来；

- 对于函数增加前缀：__declspec(dllexport)
- 更改项目属性，将生成结果该为dll；
- 点击生成，生成对应的解决方案；

```
__declspec(dllexport) void plugin(){
	// 获取到对应的内存地址
	int* p = 0x2ff9d8;
	// 修改指定内存地址的内容
	*p = 999;
}
```

#### 静态库与动态库

静态库与动态库的区别：

静态库(.a)只能使用在一个exe程序中，打包时，会被打包进行exe程序中；

动态库(.dll)能被多个程序共用，打包时不会被打进exe中，而是单独存在；
