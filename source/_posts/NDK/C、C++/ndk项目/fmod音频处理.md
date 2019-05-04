---
title: fmod音频处理
date: 2019-03-17
categories:
- C++
---
<!-- toc -->

项目下载地址：https://www.fmod.com/download

![Alt text](/images/QQ_Voicer.7z)

> 注意：

1，jniLibs文件夹下的库必须放到指定平台名称下的文件夹，否则编译时无法找到库文件。

2，对于特定平台下的库，需要在build.gradle中进行指定:
```
defaultConfig {
    ...

    ndk {
        // 因为只编译arm平台下的代码，所以必须在此进行指定
        abiFilters 'armeabi'
    }
}
```

3，库可能是C和C++混合编写的，所以最好采用C进行编译；
