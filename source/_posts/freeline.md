---
title: Freeline 在NOW上的应用
date: 2017-05-26 09:31:04
categories: Freeline
tags: Freeline
---

## 前言
随着项目的发展，引入的库越来越多，而且模块划分的也越来越多，带来的明显改变就是编译越来越来慢，代码更改之后的验证特别痛苦，严重影响开发效率。

具体的表现如下

1. 编译太慢，一般需要3~4分钟
2. 编译时，Android Studio有明显的卡顿，甚至几乎不能进行其他的操作
3. 编译好的包，在运行时经常报class not found的错误

所以去调研了一些动态的编译方案，包括:Instant Run，Buck，OkBuck，Freeline。

## 概览

| | 支持操作系统 | Gradle兼容性 | 编译速度|上手难度|问题| |
| - | - | - | - | - | - | - |
| Instant Run | Mac/Linux/Windows | 兼容 | 第一次全量编译慢，之后的增量编译较快 | 容易 | 因其代码增量是通过运行期hack method实现，所以进行了instant-run后，实际App没有重新走原有该走的生命周期，导致要看到类似onCreate，onResume等生命周期方法修改后的效果，必须手动重启一次进程，另外因为不同手机指令集合的不同，instant-run还会有一定挂掉的机会，最后，因为instant-run采用hack的方式，导致debug包调试时候无法看到对应的method堆栈，不得不说，这是个巨大的弊端，最后，instant-run不支持5.0以下的机器。 | x |
| Buck | Mac/Linux | 不兼容，需要用Buck重新写编译脚本 | 第一次全量编译慢，之后的增量编译较快 |困难 | 接入门槛高,不支持windows，不兼容Gradle | x |
|OkBuck| Mac/Linux | 兼容 | 第一次全量编译慢，之后的增量编译较快 | 一般 |虽然能兼容Gradle，但不支持Windows | x |
|Freeline | Mac/Linux/Window | 兼容 | 第一次全量编译慢，之后的增量编译较快 | 一般 | 接入麻烦一点，需要配置环境 | √ |

从上面的表格中，其实从支持的操作系统的角度来看，就只剩Freeline可以选择，因为Buck和OkBuck不支持Windows。
## Freeline简介

Freeline是蚂蚁金服旗下一站式理财平台蚂蚁聚宝团队在Android平台上的量身定做的一个基于动态替换的编译方案，稳定性方面：完善的基线对齐，进程级别异常隔离机制。性能方面：内部采用了类似Facebook的开源工具buck的多工程多任务并发思想, 并对代码及资源编译流程做了深入的性能优化。

### 优势
1. 兼容Gradle，因此我们的工程很容易引入Gradle，而且对我们的编译脚本是透明的，不会影响我们在RDM上的打包
2. 第一次全量编译和我们之前普通编译的用时是一样的，但不会感觉到卡顿
3. 之后的增量编译会很快，一般在20s左右，但当有修改AndroidManitest.xml、build.gradle、或超过20个Java文件有修改过的情况下，会自动切换到全量编译

### 劣势
1. 本地需要配置运行环境
2. 在使用过程中，发现虽然有Android Studio的插件，但插件并不是很智能，对是否该全量编译或者增量编译的情景判断不准确，例如：在手机上安装了RDM上的包，在用插件的时候，会自动启用增量编译，但手机上的包并没有启用Freeline,导致编译失败。所以，这里还需要自己手动使用命令行来编译。

### 使用

这里主要针对Windows环境进行说明

#### Python环境安装
打开cmd，依次输入下面的代码

```
@powershell -NoProfile -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
```

```
choco install python2 
```

#### Gradle脚本改造

工程跟目录的 build.gradle 添加如下:

```
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.antfortune.freeline:gradle:0.8.7'
    }
}
```

然后在Android application module 的 build.gradle 添加如下:
```
apply plugin: 'com.antfortune.freeline'

freeline {
    hack true
    productFlavor 'arm'
}

android {
    ...
}
```

最后在工程目录下，运行

```
gradlew initFreeline -Pmirror
```

> 注意:这里可能会运行失败，提示下载文件错误，这时候可以直接下载所需的文件放在工程目录下即可

#### Android Studio 上安装插件

``
File → Settings... → Plugins → Browse repositories...
``

搜 `Freeline` 安装

#### 运行
1. 点插件直接运行
2. 使用命令行
```
python freeline.py -f //全量编译
python freeline.py  //增量编译
```

## 参考

1. https://github.com/alibaba/freeline/blob/master/README-zh.md