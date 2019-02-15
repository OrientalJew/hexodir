---
title: C与Java互操作
date: 2018-07-29
tags:
- jni
categories:
- C
---
<!-- toc -->

#### JNI读取Java成员变量类型对应的签名

C访问Java中对应的成员变量，需要根据变量名和变量签名来定位变量；
![image](\images\Java数据类型签名列表.png)

<!-- more -->

对应的Java类：
```
public class JniTest {

	public String key = "hsh";

	private native static String getStringFromC();

	// 访问Java中的属性，需要为C提供访问入口
	public native String accessField();

	public static int count = 0;

	// 访问Java中的属性，需要为C提供访问入口
	public native void accessStaticField();

	public static void main(String[] args) {
//		String result = getStringFromC();
//		System.out.println(result);

		JniTest test = new JniTest();

//		System.out.println("before modify:"+test.key);
//		test.accessField();
//		System.out.println("after modify:"+test.key);

//		System.out.println("count = "+count);
//		test.accessStaticField();
//		System.out.println("count = "+count);

//		test.accessMethod();
		test.accessStaticMethod();
	}

	// 访问Java中的方法，需要为C提供访问入口
	public native void accessMethod();

	public int randomInt() {
		return new Random().nextInt();
	}

	public native void accessStaticMethod();

	public static String uuid() {
		return UUID.randomUUID().toString();
	}

	static {
		System.out.println( System.getProperty("java.library.path"));
		System.loadLibrary("jni/jni_03");
	}
}
```
#### 访问修改Java类成员变量

**C可以无视Java的权限修饰符，直接访问不公开权限的成员变量和方法；**

```
/*
	访问Java中的属性，修改其key属性的值
*/
JNIEXPORT jstring JNICALL Java_com_my_jnitest_JniTest_accessField
(JNIEnv * env, jobject obj){
	// 1、获取类
	jclass class = (*env)->GetObjectClass(env, obj);

	// 获取Java中对应属性
	// 根据属性名和Java类型的签名进行定位
	jfieldID fid = (*env)->GetFieldID(env, class, "key", "Ljava/lang/String;");

	// 获取对应的属性
	jstring jstr = (*env)->GetObjectField(env, obj, fid);

	// 将jni字符串类型转为对应的C字符串
	// 参数3指定是否以拷贝的方式进行转换，false表示在原来的内存上进行修改(实际还是进行了复制)
	// 参数3为true时，会因为无法成功复制而导致失败，所以默认使用false或者null

	/**错误纠正！！！**/
	// 此处的参数3传递的是一个jboolean类型的指针，是GetStringUTFChars函数用来
	// 告诉调用者，是否已经对字符串进行了复制备份，如果isCopy是JNI_TRUE则表示进行
	// 了复制，JNI_FALSE则表示此时操作的和Java层是同一份。
	// 也就是说isCopy相当于一个回参，用来告诉调用者，底层的操作情况；
	// 注意，默认情况下，不要修改Java层的字符串，所以当为JNI_FALSE，不要对字符串进行修改
	// 如果为JNI_TRUE，则注意，必须自己手动释放内存
	char *c_str = (*env)->GetStringUTFChars(env, jstr, NULL);

	// 使用C的方式进行修改
	char result[20] = "hello";
	strcat(result, c_str);

	//// C 字符串转为 Jni字符串
	jstring new_str = (*env)->NewStringUTF(env, result);

	// 将新值设置给Java
	// 同步回Java
	(*env)->SetObjectField(env, obj, fid, new_str);

	/**释放字符串！！！**/
	// 只要存在回参isCopy的函数，都必须我们手动释放内存！！！
	(*env)->ReleaseStringUTFChars(env, jstr, c_str);

	return jstr;
}
```

> 注意：

```
(*env)->GetStringUTFChars

```
类似于这种Getxxx方法，从Java中拷贝对应的对象，其参数3指定是否以拷贝的方式进行转换，
false表示在原来的内存上进行修改，但实际还是复制了一份到C内存中。参数3为true时，
会因为无法成功复制而导致失败，所以默认使用false或者null。
```
/**错误纠正！！！**/
// 此处的参数3传递的是一个jboolean类型的指针，是GetStringUTFChars函数用来
// 告诉调用者，是否已经对字符串进行了复制备份，如果isCopy是JNI_TRUE则表示进行
// 了复制，JNI_FALSE则表示此时操作的和Java层是同一份。
// 也就是说isCopy相当于一个回参，用来告诉调用者，底层的操作情况；
// 注意，默认情况下，不要修改Java层的字符串，所以当为JNI_FALSE，不要对字符串进行修改
// 如果为JNI_TRUE，则注意，必须自己手动释放内存

jboolean isCopy = NULL;
char *c_str = (*env)->GetStringUTFChars(env, jstr, &isCopy);

/**释放字符串！！！**/
// 只要存在回参isCopy的函数，都必须我们手动释放内存！！！
(*env)->ReleaseStringUTFChars(env, jstr, c_str);
```

在C中操作成功后，需要同步回Java中，所以可以确认，C中是拷贝了一份的。

#### 访问修改Java静态成员变量

```
JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_accessStaticField
(JNIEnv * env, jobject jobj){
  // 访问静态成员变量，通过class定位对应的类
	jclass cls = (*env)->GetObjectClass(env, jobj);
  // 定位静态变量
	jfieldID fid = (*env)->GetStaticFieldID(env, cls, "count", "I");

	jint count = (*env)->GetStaticIntField(env, cls, fid);

	count++;

	(*env)->SetStaticIntField(env, cls, fid, count);
}
```

#### C中访问Java中的成员方法

C访问Java中对应的方法，需要根据方法名和方法签名来定位方法；

获取对应的方法签名，可以cd到项目的bin目录下，通过：
```
javap -s -p com.xxx.xxx.className
```
来获取对应的方法签名；
```
JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_accessMethod
(JNIEnv * env, jobject jobj){
	jclass cls = (*env)->GetObjectClass(env, jobj);
  // 根据方法名和签名定位方法
	jmethodID mid = (*env)->GetMethodID(env, cls, "randomInt", "()I");

	// 根据Java中方法的返回值调用对应类型的方法
	jint result = (*env)->CallIntMethod(env, jobj, mid);

	printf("result : %ld", result);
}
```

#### C中访问Java中的静态方法

有些情况下，在C中想实现某个功能是非常麻烦的，比如此处生成的UUID，我们可以通过调用Java
中现成的实现方法，来方便的达到目的；

```
JNIEXPORT jstring JNICALL Java_com_my_jnitest_JniTest_accessStaticMethod
(JNIEnv * env, jobject jobj){
	jclass cls = (*env)->GetObjectClass(env, jobj);

	jmethodID mid = (*env)->GetStaticMethodID(env, cls, "uuid", "()Ljava/lang/String;");
  // 调用Java对应的方法，获取uuid
	jstring m = (*env)->CallStaticObjectMethod(env, cls, mid);
  // jstring转为char*
	char* uuid = (*env)->GetStringUTFChars(env, m, NULL);

  // 根据字符串模板构建字符串
	char* name = "";
	sprintf(name, "%s.txt", uuid);

	FILE* file = fopen(name, "w");
	fputs("hello world", file);
	fclose(file);
}
```

#### 访问Java构造方法

通过访问Java类的构造方法，可以在C中创建任意类的对象；

```
/*
	访问Java中的Date，获取当前系统时间戳
*/
JNIEXPORT jobject JNICALL Java_com_my_jnitest_JniTest_accessConstructor
(JNIEnv * env, jobject jobj){
	// 找到指定的类
	jclass cls = (*env)->FindClass(env, "java/util/Date");

	// 获取类构造方法(对于构造方法，其方法名固定为:<init>)
	jmethodID con_mid = (*env)->GetMethodID(env, cls, "<init>", "()V");

	// 实例化构造方法，创建类实例
	jobject date = (*env)->NewObject(env, cls, con_mid);

	// 根据类实例调用对应的实例方法
	jmethodID getTime_mid = (*env)->GetMethodID(env, cls, "getTime", "()J");

	jlong time = (*env)->CallLongMethod(env, date, getTime_mid);

	printf("当前时间戳:%lld", time);

	return date;
}
```

#### 绕过Java子类方法重写

在C++中，如果要重写父类的方法，需要使用virtual修饰方法，在jni中，可以利用这个特点，绕过
子类对父类相同方法的重写；

java类：
```
public class Human {

	public void sayHi() {
		System.out.println("human hi");
	}
}

public class Man extends Human {

	@Override
	public void sayHi() {
		System.out.println("man call sayhi");
	}
}

当我们在java中使用：
	private Human human = new Man();

虽然调用的是human的方法，但实际上是重新指向了Man类重写的方法；
```
C:

在JNI中，会根据调用CallNonvirtualVoidMethod方法时，传递的jclass类型，调用对应类的方
法，而不会理会Java子类的重写；

```
/*
	绕过Java的子类方法重写，调用父类的方法
*/
JNIEXPORT void JNICALL Java_com_my_jnitest_JniTest_callSuperMethod
(JNIEnv * env, jobject jobj){
	// 找到指定的类
	jclass cls = (*env)->GetObjectClass(env, jobj);

	// 在Java中，human变量被赋予了Human子类man的实例，默认情况下，调用sayHi方法
	// 会调用子类的覆盖的方法
	jfieldID fid = (*env)->GetFieldID(env, cls, "human", "Lcom/my/jnitest/Human;");

	jobject human = (*env)->GetObjectField(env, jobj, fid);
	jclass mClass = (*env)->FindClass(env, "com/my/jnitest/Human");
	jmethodID m_mid = (*env)->GetMethodID(env, mClass, "sayHi", "()V");

	// 受到Java中重写的影响，调用的是Man类的sayHi方法
	//(*env)->CallVoidMethod(env, human, m_mid);

	// 调用指定父类Human的方法
	// 如果获取到的是Man的jclass，则仍然调用的是Man的sayHi方法
	(*env)->CallNonvirtualVoidMethod(env, human, mClass, m_mid);
}
```

#### 解决Java、C字符串互传的乱码问题

在Java中，字符串一般是UTF-8编码，而在jni中，创建的字符串则是UTF-16编码，所以由C传递
给Java时，会出现乱码问题；

在C中处理字符乱码问题，过程是非常繁琐的，但是采用Java则只需要几行代码即可完成，所以，
我们可以调用Java的api来解决乱码问题；

```
JNIEXPORT jstring JNICALL Java_com_my_jnitest_JniTest_charSetProblem
(JNIEnv * env, jobject jobj, jstring str){
	// Java 传递过来的字符能够正常识别编码
	char* c_str = (*env)->GetStringUTFChars(env, str, JNI_FALSE);
	printf("%s\n", c_str);

	char* java_str = "不会算结构";
	// 默认情况下,使用的是UTF-16编码，会出现乱码
	//jstring j_str = (*env)->NewStringUTF(env, java_str);

	// 由于C中处理编码问题非常繁琐，所以可以采用Java的String类来进行处理
	jclass cls = (*env)->FindClass(env, "java/lang/String");
	// 调用String类的构造方法：String(byte[], charset)
	jmethodID mid = (*env)->GetMethodID(env, cls, "<init>", "([BLjava/lang/String;)V");

	// C中，char对应着Java中的byte
	jbyteArray ba = (*env)->NewByteArray(env, strlen(java_str));

	// 转化成的字符编码
	char* result_charset = "GB2312";
	jstring charset = (*env)->NewStringUTF(env,result_charset);

	// jni的jbyte对应C的 signed char 类型，所以可以直接将char* 赋给jbyteArray
	(*env)->SetByteArrayRegion(env, ba, 0, strlen(java_str), java_str);

	// 调用String对应的构造方法，构造出正确编码的字符串
	jstring result = (*env)->NewObject(env, cls, mid, ba, charset);
	return result;
}
```

#### C与Java数组交互

从Java中拷贝对应的数组到C的内存中，操作后，再同步回Java数组中；

C代码：
```
JNIEXPORT jstring JNICALL Java_com_my_jnitest_JniTest_sortArray
(JNIEnv * env, jobject jobj, jintArray arr){
	// 获取C的jint数组 (java中的int对应C中long，不能直接用C的int接收)
	jint* c_arr = (*env)->GetIntArrayElements(env, arr, JNI_FALSE);

	// 获取数组长度
	int len = (*env)->GetArrayLength(env, arr);

	// 排序
	qsort(c_arr, len, sizeof(jint), compare);

	// 同步
	// 0, 同步Java数组，释放C数组
	// 1, JNI_COMMIT 同步Java数组，不释放C数组(函数执行完成会自动释放)
	// 2, JNI_ABORT  不同步Java数组，同时释放C数组
	(*env)->ReleaseIntArrayElements(env, arr, c_arr, 0);
}
```

> 注意：

```
(*env)->GetIntArrayElements

```
类似于这种Getxxx方法，从Java中拷贝对应的对象，其参数3指定是否以拷贝的方式进行转换，
false表示在原来的内存上进行修改，但实际还是复制了一份到C内存中。参数3为true时，
会因为无法成功复制而导致失败，所以默认使用false或者null。

在C中操作成功后，需要同步回Java中，所以可以确认，C中是拷贝了一份的。

Java代码：
```
	public native void sortArray(int[] arr);

	int[] arr = new int[]{3,8,1,5,4};
	// 调用Native方法
	test.sortArray(arr);
	for(int a:arr) {
		System.out.println(a);
	}
```

##### 向Java返回数组

jni所代表的数组与C中操作的数组分属两个不同的内存中，所以在C中操作完成之后，结果需要
同步进jni数组中，才能够影响到Java层；

```
// 获取C数组
JNIEXPORT jstring JNICALL Java_com_my_jnitest_JniTest_getCArray
(JNIEnv * env, jobject jobj, jint len){
	// 在内存上，jintArray和jint* 两个数组是处于不同的内存中
	jintArray jint_array = (*env)->NewIntArray(env, len);
	jint* array = (*env)->GetIntArrayElements(env, jint_array, NULL);

	// 获取数组长度
	int alen = (*env)->GetArrayLength(env, jint_array);

	for (int i = 0; i < alen; i++){
		array[i] = i;
	}

	// 上面只是操作了C所在部分的数组，此处需要同步到jintArray中
	(*env)->ReleaseIntArrayElements(env, jint_array, array, 0);
	return jint_array;
}
```
