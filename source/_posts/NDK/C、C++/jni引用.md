---
title: jni中的引用
date: 2018-10-07
categories:
- C++
---
<!-- toc -->

#### jni中的引用

jni引用类型分为：局部引用和全局引用

作用：在JNI中告知虚拟机何时回收一个JNI变量
<!--more-->
#### 局部引用

局部引用最大的特点就是能够让我们边使用，边释放，保证小的的内存消耗；

比如，访问较大的Java对象(Bitmap、Array)，使用完后，还要进行其他复杂的耗时操作，此时可以
先将对象回收；或者，当方法中创建了大量的局部引用，占用了太多的内存，而且这些局部引用跟后
面的操作没有关联，也可以直接回收掉；

```
JNIEXPORT jstring JNICALL Java_com_my_jnitest_JniTest_releaseCRef
(JNIEnv * env, jobject jobj){
	int i = 0;
	for (; i < 5; i++){
		jclass cls = (*env)->FindClass(env, "java/util/Date");
		jmethodID method_id = (*env)->GetMethodID(env, cls, "<init>", "()V");
		jobject obj = (*env)->NewObject(env, cls, method_id);

		//...

		// 对于不再使用的对象，通过垃圾回收期主动释放局部引用
		(*env)->DeleteLocalRef(env, obj);

		//...
	}
}
```

#### 全局引用

全局引用可以被全局多个函数创建、使用和释放，所以使用之前必须先做判空处理；

全局引用是可以被多个线程共享的，但需要注意线程安全问题；

```
// 创建全局引用
// 创建共享引用(可以跨多个线程使用)，灵活控制内存的使用
jstring global_str;

JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_createGlobal
(JNIEnv * env, jobject jobj){
	// 创建全局引用，必须通过包裹局部引用来完成
	jstring obj = (*env)->NewStringUTF(env, "jni is powerful!");
	global_str = (*env)->NewGlobalRef(env, obj);
}

JNIEXPORT jstring JNICALL Java_com_my_jnitest_JniTest_getGlobal
(JNIEnv * env, jobject jobj){
	return global_str;
}

JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_deleteGlobal
(JNIEnv * env, jobject jobj){
	(*env)->DeleteGlobalRef(env, global_str);
}
```

#### 弱全局引用

弱全局引用其实与Java中的弱引用类似，在垃圾回收时，如果内存不足，则引用对象会被回收；

通过弱全局引用，可以达到灵活创建临时变量，当检测到为NULL时，我们再次再创建；
```
jstring weak_global_str;

JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_createWeakGlobal
(JNIEnv * env, jobject jobj){
	// 创建全局引用，必须通过包裹局部引用来完成
	jstring obj = (*env)->NewStringUTF(env, "jni is powerful!");
	global_str = (*env)->NewWeakGlobalRef(env, obj);
}
```
