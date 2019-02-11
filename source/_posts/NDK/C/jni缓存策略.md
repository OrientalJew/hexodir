---
title: jni缓存策略
date: 2019-2-11
tags:
- jni
categories:
- C
---
<!-- toc -->

### 通过局部静态变量进行缓存

通过在方法中声明局部的静态变量，可以在方法初始化时进行方法中的某个资源的初始化，
并且在以后每次调用该方法时，确保不需要重复初始化该资源，同时确保其他的方法无法调用
到。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "jni.h"

// 缓存策略

JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_cached
(JNIEnv * env, jobject obj){
	jclass cls = (*env)->GetObjectClass(env, obj);

	// 静态局部变量
	// 作用域存在于本函数
	// 生命周期为应用生命周期
	// 全局的其他地方无法使用到
	static jfieldID key_id = NULL;
	// 只在函数被第一次调用时进行初始化
	if (key_id == NULL){
		key_id = (*env)->GetFieldID(env, cls, "key", "Ljava/lang/String;");
		printf("init key id;");
	}
}
```

***Java中调用***

```
	public native void cached();

  for (int i = 0; i < 5; i++) {
    test.cached();
  }
```

### 全局变量初始时进行创建

全局变量可以在dll库加载时进行进行全局变量的初始化；

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "jni.h"

// 初始化全局变量
// 在初始化动态库之后，调用该方法，进行全局变量的初始化，并且能够确保只调用一次。
jmethodID method_id;
jfieldID fid;
JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_initIds
(JNIEnv * env, jclass jcls){
	method_id = (*env)->GetMethodID(env, jcls, "randomInt", "()I");
	fid = (*env)->GetFieldID(env, jcls, "key", "Ljava/lang/String;");
	printf("first init!");
}
```

***Java中调用***

```
static {
  System.out.println( System.getProperty("java.library.path"));
  System.loadLibrary("jni/jini_04");
  // 加载完成动态库之后，立刻调用初始化方法，初始化全局变量
  initIds();
}
```
