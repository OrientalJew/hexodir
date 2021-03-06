---
title: jni异常处理
date: 2018-10-07
categories:
- C++
---
<!-- toc -->

#### jni异常处理

jni层抛出的异常是无法在Java层被捕获到，只能在C层进行清空，如果有必要，需要向Java层主动
抛出异常进行警告；

/**错误纠正：jni层的异常是可以被Java层捕获到的！！！**/
// 根据Jni层抛出的异常，可以发现其实Throwable和Error的子类，所以我们可以通过捕获这两者
// 来达到捕获的目的
```
JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_catchException
(JNIEnv * env, jobject jobj){
	jclass cls = (*env)->GetObjectClass(env, jobj);
	jfieldID fid = (*env)->GetFieldID(env, cls, "key2", "Ljava/lang/String;");

	// 捕获Jni异常
	jthrowable exc = (*env)->ExceptionOccurred(env);
	if (exc != NULL){
		// 清空异常，确保Java层能够运行
		(*env)->ExceptionClear(env);

		// 重新获取
		fid = (*env)->GetFieldID(env, cls, "key", "Ljava/lang/String;");

		jstring str = (*env)->GetObjectField(env, jobj, fid);
		char* c_str = (*env)->GetStringUTFChars(env, str, NULL);

		// 抛出Java层能够捕获的异常
		if (_stricmp(c_str, "hsh") != 0){
			jclass new_exc = (*env)->FindClass(env, "java/lang/IllegalArgumentException");
			(*env)->ThrowNew(env, new_exc, "key's value is invalid!");
		}
	}
}
```

java:
```
try {
  // Jni层次的异常是无法被捕获到的
  test.catchException();
} catch (Exception e) {
  // 我们在Jni中主动抛出的异常能够被捕获
  System.out.println(e.getMessage());
}
```
/**通过捕获Throwable进行达到Jni层的异常捕获**/
```
try {
  // Jni层次的异常是无法被捕获到的
  test.catchException();
} catch (Throwable e) {
  // 我们在Jni中主动抛出的异常能够被捕获
  System.out.println(e.getMessage());
}
```
