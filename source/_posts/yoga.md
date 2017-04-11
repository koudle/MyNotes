---
title: Facebook YOGA初探:跨平台的布局引擎(基于Android)
date: 2017-04-11 21:13:50
categories: Android
tags: Android
---

![](http://upload-images.jianshu.io/upload_images/523749-1e53924c53cfec6b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# YOGA简介
Facebook引领着移动开源风向，这次它对布局出手了，推出了Yoga开源项目，意在打造一个跨iOS、Android、Windows平台在内的布局引擎，兼容Flexbox布局方式，让界面布局更加简单。

Yoga官网：https://facebook.github.io/yoga/

官网上描述的特性包括：

* 完全兼容Flexbox布局，遵守W3C的规范

* 支持Java、C#、Objective-C、C四种语言

* 底层代码使用C语言编写，性能不是问题

* 支持流行框架如React Native

## 背景
目前，各个平台都有自己的一套解决方案。iOS平台有AutoLayout，Android有容器布局系统，而Web端有基于CSS的布局系统。多种布局系统共存所带来的弊端是很明显的，平台间的共享变得很困难，而每个平台都需要专人来开发维护，增加了开发成本。

像微信小程序、ReactNative这种，都是采用js/html/css开发，客户端native展现的方式，那如何把前端的布局对应到Android/iOS的不同平台上？

微信小程序、ReactNative都是怎么来实现的？

YOGA就是为了解决这样的问题产生的。

在React Native里Facebook里引入了一种跨平台的基于CSS的布局系统，它实现了Flexbox规范。基于这个布局系统，不同团队终于可以走到一起，一起解决缺陷，改进性能，让这个系统更加地贴合Flexbox规范。随着这个系统的不断完善，Facebook决定对它进行重启发布，并取名Yoga。

Yoga是基于C实现的。之所以选择C，首先当然是从性能方面考虑的。基于C实现的Yoga比之前Java实现在性能上提升了33%。其次，使用C实现可以更容易地跟其它平台集成。到目前为止，Yoga已经有以下几个平台的绑定：Java（Android）、Objective-C（UIKit）、C#（.NET）。而且已经有很多项目在使用Yoga，比如React Native、Components for Android、Oculus，等等。

不同于其它的一些布局框架，比如bootstrap的栅格系统或Masonry，它们要么不够强大，要么不支持跨平台。Yoga遵循了Flexbox规范，同时又将布局元素抽象成Node，为各个不同平台暴露出一组标准的接口，这样不同的平台只需实现这些接口就可以了。

当然，Facebook不会就此止步。作为一款跨平台的布局引擎，自然需要各个平台的开发人员一起努力来促进它的发展，所以Facebook把Yoga开源了。目前微软已经成为Yoga的贡献者之一，他们不仅修复缺陷，还为Yoga带来新的特性。

除了完全遵循Flexbox规范，Facebook还计划在未来为Yoga加入更多特性，这些特性将超出Flexbox的范畴。


## 应用展望

从目前为止，困扰客户端的一大问题就是Android/iOS完全不同的布局系统，对于Android来说复杂的布局必须使用xml来描述，而xml不能动态发布的特性，也完全限制了客户端view的展现，试想一下，如果能把Android/iOS的布局统一，并且可以根据下发的布局配置实时生成动态的布局，岂不是很爽！

利用YOGA我们可以
* 只写一次布局，就可以得到在不同端上的布局展示。
* 动态更改view的布局
* 还有各种意义。。。

总的来说，大前端的趋势越发明显

## YOGA初探

当得知YOGA开源的消息时，楼主第一时间去github上看了源代码，Facebook提供了iOS和Windows的sample工程，唯独没有提供Android的example，只有一个Java库的代码，于是楼主在想既然源码都有，那写一个Android的example，于是就动起手来，最后终于把YOGA集成在了Android的工程上，并顺利跑通。具体过程如下。

### 生成so
YOGA是基于C实现的，因此我们的思路很明确，就是把YOGA的C代码交叉编译成可在
Android上运行的so。这里我们需要用到[BUCK](https://buckbuild.com/),因为YOGA是基于BUCK构建的，BUCK可以理解成是和Gradle一样的构建工具，这里有一个关于Gradle和Buck的构建速度对图：

![](http://upload-images.jianshu.io/upload_images/523749-c5157f978bea0fe0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



要得到android-armv7的so，需要交叉编译，我们需要确保本地已经安装了Android NDK的环境，可以参考[Android 官网](https://developer.android.com/ndk/index.html)。

通过分析YOGA工程的BUCK配置，发现YOGA由3个模块组成，分布在三个地方，分别是:
* libyogacore.so 在/yoga目录下
* libyouga.so 在/java/jni目录下
* liblib_fb_fbjni.so 在/lib/fb 目录下

我们只要执行BUCK的构建代码，就可以生成上面提到的三个so，构建代码如下:
```
$ buck build //java:jni#android-armv7,shared
```

这里有坑，编译时我们需要两个flavor，分别是`android-armv7`和`shared`
* android-armv7 就是支持armeabi架构的.so文件，这里需要有NDK环境的支持
* shared 就是生成动态链接库

因为，如果我们直接使用默认的构建配置
```
$ buck build
```
会发现，生成的不是动态链接库so，而是静态链接库，而且也不是基于android-armv7生成的so，所以需要指定编译的platform和明确要编译的是动态链接库，才能生成可用的so.

当完成上面的步骤之后，我们还会发现一个问题，就是当你的工程在加载so的时候，会报错：
```
can't find  libgnustl_shared.so 
```
这时我们需要把ndk目录下的 libgnustl_shared.so 也拷贝到工程目录下，路径如下：
```
android-ndk-r13b/sources/cxx-stl/gnu-libstdc++/4.9/libs/armeabi-v7a
```

为什么会有这样的
### 创建Android工程
在生成so之后，后面的步骤大家应该很清楚了，把生成的so放在Android目录下的jniLibs下，并把YOGA中的java库引入工程，一个完整的可以运行YOGA的Android工程就创建好了。

### sample
目前写了一个很简单的用YOGA实现的布局，首先我们用基于Flexbox的规范来布局view，然后调用`` root.calculateLayout()`` 就可以得到在Android下的view的坐标、长、宽等基本布局信息,代码如下：

```
            YogaNode root = new YogaNode();
            root.setWidth(500);
            root.setHeight(300);
            root.setAlignItems(YogaAlign.CENTER);
            root.setJustifyContent(YogaJustify.CENTER);
            root.setPadding(YogaEdge.ALL, 20);

            YogaNode text = new YogaNode();
            text.setWidth(200);
            text.setHeight(25);

            YogaNode image = new YogaNode();
            image.setWidth(50);
            image.setHeight(50);
            image.setPositionType(YogaPositionType.ABSOLUTE);
            image.setPosition(YogaEdge.END, 20);
            image.setPosition(YogaEdge.TOP, 20);

            root.addChildAt(text, 0);
            root.addChildAt(image, 1);

            root.calculateLayout();

            Log.d(TAG,"text,layout X:"+text.getLayoutX()+" layout Y:"+text.getLayoutY());
            Log.d(TAG,"image,layout X:"+image.getLayoutX()+" layout Y:"+image.getLayoutY());
```

### 工程
目前我把基于YOGA的Android工程已传到github，欢迎各位多多交流，多给star。Thanks!
地址:https://github.com/koudle/AndYogaSample


#### 文章参考:
https://facebook.github.io/yoga/
https://buckbuild.com/
http://mp.weixin.qq.com/s/VT6uytG8di-KNfJDCZE5Og