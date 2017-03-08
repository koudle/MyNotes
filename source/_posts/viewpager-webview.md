---
title: 记一次需求的开发--在ViewPager中嵌套WebView
categories: Android
tags: Android
---

###  前言
 ViewPager是可以左右滑动的组件，我们经常会用到，当ViewPager中嵌套了WebView且WebView中的内容也需要左右滑动的时候就会出现事件冲突：因为ViewPager需要消费左右滑动的事件，WebView也需要消费左右滑动的事件，那么该如何解决这样的问题？首先我们来回顾下Android事件传递的机制。
 
### Android事件传递机制
其实网上有很多讲解Android事件传递机制的文章，但感觉讲的都不是很清楚，因为网上大部分都是切割成三部分来讲，分别为：Activity、ViewGroup、View，但其实Android事件传递本质上是一个递归，如果单纯的切割开来，会忽略很多的内部实现细节，所以都不如自己看源代码来的实在。

我们从事件开始传递的最初入口开始讲起
### Activity
事件是从Activity开始传递，具体代码如下：
`Activity.java`
```
    public boolean dispatchTouchEvent(MotionEvent ev) {
    	//如果事件为ACTION_DOWN，则调用onUserInteraction(),我们可以看到这个函数的实现为空
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        //调用window的dispatchTouchEvent，如果返回true，即下层的消费了此事件，则直接return true，否则返回false，即下层没有消费此事件，就掉用Activity的onTouchEvent消费本次事件
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```

在看Window里的superDispatchTouchEvent是怎么实现的
这里的getWindow返回的winow是PhoneWindow的实例
`PhoneWindow.java`
```
    @Override
    public boolean superDispatchKeyEvent(KeyEvent event) {
    	//这里调用的是mDecor的superDispatchKeyEvent
        return mDecor.superDispatchKeyEvent(event);
    }
```

mDecor是DecorView的实例
`DecorView.java`
```
public boolean superDispatchTouchEvent(MotionEvent event) {
       //这里是调用的父类的dispatchKeyShortcutEvent方法
        return super.dispatchTouchEvent(event);
    }
```
DecoreView的父类是FrameLayout,FrameLayout没有实现dispatchTouchEvent，因此调用的是FrameLayout的父类ViewGroup的dispatchTouchEvent的方法。

第一部分讲到这就结束了，因为我们都知道Activity的最外层包裹的是DecorView，DecoreView里面在包裹我们自己定义实现的Activity的view，而DecoreView继承自FrameLayout，本质上也是一个view，所以总结来说就是Activity获得了事件，随即就抛给了view来处理。

