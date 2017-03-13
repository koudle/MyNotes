---
title: 一个步骤教你调试Android系统源代码
categories: Android
tags: Android
---
有时候我们为了搞懂Android系统组件的运行原理，需要查看系统的源代码，但是如果我们不仅能看源码，要是还能调试，岂不是更好更方便。

所以，我们就说下如何调试系统源代码。其实很简单的了。

#### 1.确认自己手机的API Level
这个很简单了，就是确定自己手机系统的版本号，然后下载对应的源码，如果手机系统的版本和源码的版本对不上，那么debug的时候很容易出现对应不上代码行数的问题，因为每个版本的源码都可能有更新。

#### 2.用Android Studio自带的SDK Manager下载对应版本的源码
"Tools" --> "Android" --> "SDK Manager"
如下图：

{% asset_img sdk_manager.png SDK Manager%}

* 注意
```
1. 右下角的 "Show Package Details" 请勾选上
2. 在Android 7.1.1 (Nougat)下面有一个选项 "Sources For Android 25" ，这个就是我们要下载的源码，请选中下载
```

#### 3.还有最重要的一点，compileSdkVersion
compileSdkVersion,是用来告诉Gradle用哪个Android  SDK版本来编译APK，所以这里的compileSdkVersion也必须和第一步中你手机的API Level保持一致，否则，你在IDE上用到的Android SDK的源码和你的手机系统的版本不一致，就会出现调试的代码行数对不上的问题。

这里还有一点说明一下，当你查看源码的时候，有这样一个路径，如下图：

{% asset_img right.png android-25 viewGroup的源码 %}
这里我们的compileSdkVersion是25,所以查看的源码路径上是/android-sdk-linux/sources/`android-25`，注意这里的android-25，而且ViewGroup.java显示的也是普通java文件的样子，现在给一个错误的例子，我们选择android-23的ViewGroup.java，显示的却是下图：
{% asset_img error.png android-23 viewGroup的源码 %}
很容易看到不同的地发

#### 4.最后一部，加上断点，attach上进程，就可以愉快的调试源码了
这里和我们调试普通代码就是一样的了。经过上面的操作，Android源码就像我们添加的一个库一样。

#### 补充
刚开始我是用Nexus 6P来调试的，因为是谷歌的亲儿子，后来用同样的方法试了华为的手机，发现也可以，所以虽然一些第三方ROM有很多更改的地方，但对一些基本的FrameWork应该是没啥改动的，改的更多的应该是桌面啊这些，但这些已经属于Application层了，现在给张Android 体系结构图。

{% asset_img android.jpg Android 体系结构图 %}