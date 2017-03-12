---
title: 用3个View实现无线循环的ViewPager
categories: Android
tags: Android
---

## 前言
用ViewPager实现无限循环，想必大家都知道，但实现无线循环的效果，其实只需要三个view便能完成，本篇就先介绍下实现无限循环的原理，在告诉大家怎么只用三个view边实现无限循环。

### LoopViewPager的原理
下面有四个View：A、B、C、D，默认情况下我们只能从A->B->C->D或者从D->C->B->A，如下图：

{% asset_img origin.png  %}

每当移动到两边的view时，就没办法在滑动下去了，而要实现无限循环滑动，必须得实现这样的效果，A->B->C->D->A,或者D->C->B->-A->D,所以我们出现了方案1如下图：

{% asset_img modify.png 方案一%}

当在D上往左拖动的时候，就定位到A，而当A往右拖动的时候就定位到D，但是这个方案有个巨大的缺陷，当D往左拖动的时候，D的右边是没有view的，这样就不会有拖动中view的动画效果，所以这个方案pass。

继续上面的思路想，那往D的右边加一个和A一样的view（我们暂时称为A'），这样当D往左拖动的时候，显示的是A'，当拖动动画结束的时候，定位到A（PS：这样里用ViewPager.setCurrentItem(A,false)来实现，因为是false，所以不会展现动画，用户就感知不到这里的变化），这里用setCurr这样就又从头开始，这个方法可行，效果图如下：

{% asset_img modify2.png 方案二%}

具体的实现代码请看： https://github.com/imbryk/LoopingViewPager
这个代码就是按照方面二的思路来实现的，这里的例子给的是4个view，之后不管我们要实现多少个view的无限循环，只要我们在队首加一个最后的view，在队尾加一个第一个的view，我们都能实现无限循环。

到这里就结束了？当然没有，因为我们的目标是不管你要展示多少个view，我们只用三个view就可以实现全部view的无限循环！

### 三个View实现无限循环
先看图：

{% asset_img final.png 三个View实现无限循环%}

我们可以看到这个方案其实和上面的方案二很像，不同的是我们用到了view的复用，而且我们给用户展现的永远是第二个view，每当用户有滑动操作的时候，我们就更新
第二个view的数据，以此来更新试图，让用户误以为自己是在无限滑动

具体的代码可以查看我的工程，欢迎star，github地址为： https://github.com/koudle/ThreeInfiniteViewPager