---
title: Android在Ubuntu上配置NDK的环境
categories: Android
tags: Ubuntu,Android
---
由于自己一直在使用Ubuntu，所以Android的开发环境也在Ubuntu上搭建，在开发过程中，需要用到Android的交叉编译，所以把Android在Ubuntu上配置NDK环境的步骤记录一下。
## 1.下载Android NDK压缩包
[官网下载](https://developer.android.com/ndk/downloads/index.html)
NDK有不同的版本，这里根据需要下载不同的版本。
## 2.解压
把第1部下载的压缩包解压，我这里存放的目录如下：
```
/home/kl/android-sdk-linux/android-nkd-r13b
```
## 3. 设置环境变量
执行如下命令
```
$ sudo vim ~/.bashrc
```
然后经如下几句加进去：
```
export ANDROID_HOME="/home/kl/android-sdk-linux"
export PATH="$PATH:$ANDROID_NDK"
```
关闭terminal，在打开就设置成功了，在验证一下是否成功：
```
$ ndk-build -v
GNU Make 3.81
Copyright (C) 2006  Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.

This program built for x86_64-pc-linux-gnu
```