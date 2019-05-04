---
title: C++ windows 下使用FFmpeg
date: 2019-03-31
categories:
- C++
---
<!-- toc -->

#### 下载编译库

地址：https://ffmpeg.zeranoe.com/builds/

> Shared版中已经有编译好的dll动态库，可以直接拿来用。
Dev版本为源码，代码中需要调用;

![image](/images/ffmpeg1.jpg)

#### 配置项目

> 添加include文件夹

![image](/images/ffmpeg2.jpg)

> 添加lib文件夹

![image](/images/ffmpeg4.jpg)


> 增加依赖库

```
avcodec.lib
avformat.lib
avutil.lib
avdevice.lib
avfilter.lib
postproc.lib
swresample.lib
swscale.lib
```
![image](/images/ffmpeg3.jpg)

> 因为我们下载的项目是64位的平台，所以需要配置为x64

![image](/images/ffmpeg5.jpg)

#### 测试配置是否成功

```
#include<iostream>

// C和C++进行混编，以C的方式进行编译
extern "C"{
#include"libavcodec\avcodec.h"
}

using namespace std;

void main(){
	cout << avcodec_configuration() << endl;

	// 解码，编码，播放...

	system("pause");
}
```
