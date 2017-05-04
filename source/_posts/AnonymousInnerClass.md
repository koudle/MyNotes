---
title: 危险的匿名内部类系列之-----Runnable
date: 2017-05-04 21:56:39
categories:
tags:
---

## 分析
||优势|危险|
|-|-|-|
|匿名内部类（AnonymousInnerClass）|简化代码|匿名内部类本身会对外部类有一个引用（this引用）|

## 容易出问题的地方
Runnable是我们常用的类，主要与handler配合使用，可以指定在主线程或子线程使用，也可以用于延迟任务。

这里我们要警惕延迟任务，尤其是匿名内部类Runnable用在其他类的handler上的延迟任务，因为handler会一直hold住我们的延迟的Runnable任务，而匿名内部类的Runnable含有对外部类的this引用，会导致内存泄漏

## 解决方法
如果我们的应用里存在上面描述的常见，那么最好是一个类实现一个handler，当这个类销毁的时候，把对应的handler也销毁掉。