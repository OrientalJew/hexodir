---
title: 开源C/C++库使用
date: 2019-02-16
tags:
- jni
categories:
- C
---
<!-- toc -->

#### bsdiff增量更新库使用步骤以及问题处理

1，下载bsdiff源码：(bsdiff4.3-win32-src.zip)
http://www.daemonology.net/bsdiff/

2，创建一个新的C项目，将源码中的C和C++文件拷贝进来；
```
暂时只需要bsdiff.cpp，用来拆分新包，bspatch.cpp不可以引入；
bspatch是用来合并包文件的，需要引入bzip2的源码来辅助操作，如果引入将会报错！！！
```

3，关于_CRT_SECURE_NO_WARNINGS 错误：
```
由于项目中可能使用了老版本、非安全版本的函数，所以编译时将会报错，解决方案是继续使用这些
函数。

另一个问题是，如果存在多个这种情况的源文件时，可以通过在命令行中输入：
-D _CRT_SECURE_NO_WARNINGS
```
![image](\images\c非安全函数警告.jpg)

4，报错：warning C4996: 'setmode': The POSIX name for this item is deprecated. Instead, use the ISO C++ conformant name: _setmode. See online help for details.

```
原因与上面类似，可以通过增加宏定义来消除警告:_CRT_NONSTDC_NO_DEPRECATE
```

![image](\images\C安全警告.jpg)


5，error C4703: 使用了可能未初始化的本地指针变量“_new”

```
由于源码使用了不同系统的编译器，而visual studio默认对语法进行了严格的检查，所以会出现以上的警告，解决方案是关闭掉vs的检查；
```
![image](\images\关闭语法检查.jpg)

> 如果生成解决方案成功，则说明配置成功，可以开始进行jni配置；

6，可以看到在bsdiff.cpp中存在启动main函数，通过调用该方法可以生成我们需要的拆分包；

```
// 将main函数修改为bsdiff_main，方便我们的jni函数调用
int bsdiff_main(int argc,char *argv[]){
  //...

  // 可以看到，此处要求我们传入四个参数
  // 参数1：任意
  // 参数2：旧apk文件的路径
  // 参数3：新apk文件的路径
  // 参数4：拆分出来的文件保存路径
  if(argc!=4) errx(1,"usage: %s oldfile newfile patchfile\n",argv[0]);

}
```

7，编写jni函数，并生成对应的解决方案：
```
JNIEXPORT void JNICALL Java_com_hsh_bsdiff_BsdiffUtill_diff_1apk
(JNIEnv *env, jclass jcls, jstring fileName_, jstring oldFile_, jstring newFile_, jstring patchFile_){

	char* fileName = (char*)env->GetStringUTFChars(fileName_, 0);
	char* oldFile = (char*)env->GetStringUTFChars(oldFile_, 0);
	char* newFile = (char*)env->GetStringUTFChars(newFile_, 0);
	char* patchFile = (char*)env->GetStringUTFChars(patchFile_, 0);


	char *args[] = { fileName, oldFile, newFile, patchFile};

	bsdiff_main(4, args);

	env->ReleaseStringUTFChars(fileName_, fileName);
	env->ReleaseStringUTFChars(oldFile_, oldFile);
	env->ReleaseStringUTFChars(newFile_, newFile);
	env->ReleaseStringUTFChars(patchFile_, patchFile);
}
```

对应的Java方法：

```
public class BsdiffUtill {

	static{
		System.loadLibrary("jni/patch_dispatch_apk");
	}

	/**
	 *
	 * @param name
	 * @param oldFile
	 * @param newFile
	 * @param patchFile
	 */
	public native static void diff_apk(String name,String oldFile,String newFile,String patchFile);
}


public class Bsdiff {

	public static void main(String[] args) {
		BsdiffUtill.diff_apk("apk_patch", "H:\\bsdiff_patch\\app_1.0version.apk",
				"H:\\bsdiff_patch\\app_2.0version.apk", "H:\\bsdiff_patch\\app_patch.patch");
	}
}

```
运行Java程序，可以在对应路径下生成拆分包；

8，合并拆分包；

_注意，Android是linux系统，所以在编写NDK时，bsdiff要使用linux系统版本下的源码。_

9，bsdiff合并时，需要使用到bzip2的源码，所以同样需要下载bzip2的源码：
https://download.csdn.net/download/juncojet/10235311

然后引进项目中；

10，让bspatch.c中引用bzip2的源码；

添加头文件引用：
```
// 本源文件只要求引入该头文件
#include "bzip2/bzlib.c"

// 但实际编译时，缺报bzlib.c缺少对应的引用函数(BZ2_crc32Table,BZ2_compressBlock,BZ2_crc32Table...)，
// 大概猜测原因是：在配置CMakeLists时，只配置了bspatch.c文件，而bzip2源文件并没有配置，所以这些文件没有
// 被项目管理起来。在bzlib.c中，头文件bzlib_private.h声明的函数无法被项目知道，所以在bzlib.c进行调用时，
// 才会报出找不到这些函数的错误。解决方案就是引入底下的源文件，不再通过bzlib_private.h找，而是直接引用；
#include "bzip2/crctable.c"
#include "bzip2/compress.c"
#include "bzip2/decompress.c"
#include "bzip2/randtable.c"
#include "bzip2/blocksort.c"
#include "bzip2/huffman.c"
```
> 之后按照上面的生成拆分包的步骤，编写Android下的NDK；

11，在bspatch.c下添加jni调用函数：
```
JNIEXPORT void JNICALL
Java_com_hsh_bsdiff_1patch_PatchUtil_patchFile(JNIEnv *env, jclass type, jstring oldFile_,
                                               jstring newFile_, jstring patchFile_) {
    char *oldFile = (char *) (*env)->GetStringUTFChars(env, oldFile_, 0);
    char *newFile = (char *) (*env)->GetStringUTFChars(env, newFile_, 0);
    char *patchFile = (char *) (*env)->GetStringUTFChars(env, patchFile_, 0);

    char *fileName = "apk_patch";

    char *params[] = {fileName, oldFile, newFile, patchFile};
    // 将源文件的main函数直接修改为patch_main，进行调用
    patch_main(4,params);

    __android_log_print(ANDROID_LOG_INFO, "PatchFile", "oldFile = %s", oldFile);
    __android_log_print(ANDROID_LOG_INFO, "PatchFile", "newFile = %s", newFile);
    __android_log_print(ANDROID_LOG_INFO, "PatchFile", "patchFile = %s", patchFile);

    (*env)->ReleaseStringUTFChars(env, oldFile_, oldFile);
    (*env)->ReleaseStringUTFChars(env, newFile_, newFile);
    (*env)->ReleaseStringUTFChars(env, patchFile_, patchFile);
}
```

Java代码：
```
public class PatchUtil {

    static{
        System.loadLibrary("native-lib");
    }

    public native static void patchFile(String oldFile,String newFile,String patchFile);
}
```

12，旧版的apk中需要有合并升级的逻辑，这样在下载拆分包后，可以直接进行合并；

**重要的api**
```
// 获得应用包下保存的当前版本的apk
final String apkPath = getApplicationContext().getPackageResourcePath();
```

模拟合并后执行安装：
```
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            PatchUtil.patchFile(apkPath, newPath, patch_path);
            runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(MainActivity.this, "success", Toast.LENGTH_LONG).show();
                    // 7.0以上需要配置FileProvider才能安装
                    ApkUtils.installApk(MainActivity.this, newPath);
                }
            });
        } catch (Error error) {
            error.printStackTrace();
        }
    }
}).start();
```
**生成新的apk文件的路径可以是任意，即使在安装后被删除也是无所谓的，因为应用包下会自动保有原来的apk文件！**

> 当有新版的apk发布时，利用新版的apk与旧版的apk，调用上面步骤生成拆分库，生成对应的拆分包。

> 不同版本的旧apk通过访问服务器，获取到对应的升级拆分包，进行升级。

*差分算法*

- 差分算法是对新旧的两个文件进行比较，筛选，分类出两个文件中的相同部分，不同部分(修改的部分)，以及额外增加的部分；

- 对于相同的部分，会用下标标识出来，对于不同的部分和额外增加的部分，会通过bzip2进行压缩，
所以生成的差分包 = 映射标识相同位置的数据+不同部分、额外增加部分的压缩数据；

- 合成时，对于相同的部分，根据差分包中标识的下标，从旧文件中直接读取出来，写进新文件中；对于不同的部分(修改的部分)和额外增加的部分，通过bzip2进行解压，再写进新文件中；

特点，生成差分包时，差分包的大小并不与新、旧包大小差值成正比关系，而是与新旧包中数据的重复情况成正比关系：*重复的数据越多，则生成的差分包越小；被修改的部分越多，则对应生成的映射标识越多，差分包也越大；*  
