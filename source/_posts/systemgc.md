---
title: systemgc
categories:
tags:
---
## 导语
曾几何时，我们一直纠结于到底要不要手动调用System.gc()，有的人说这样调用太丑陋，完全没必要，JVM会帮我们处理好的，有的人说可以调用，这样可以及时释放内存，现在可以明确的告诉你，在Android5.0及以上手动调用System.gc()完全没必要，因为你调了也完全触发不了gc，为什么会这样说呢？

`Talk is cheap,show you the code`

我们直接看代码!


## 源码

### Android5.0之前的代码

如下:

    /**
     * Indicates to the VM that it would be a good time to run the
     * garbage collector. Note that this is a hint only. There is no guarantee
     * that the garbage collector will actually be run.
     */
    public static void gc() {
        Runtime.getRuntime().gc();
    }
	
这个看上去完全没问题

### Android 5.0及以后的代码

先看System.gc()到底做了什么，如下:


    /**
     * Indicates to the VM that it would be a good time to run the
     * garbage collector. Note that this is a hint only. There is no guarantee
     * that the garbage collector will actually be run.
     */
    public static void gc() {
        boolean shouldRunGC;
        synchronized(lock) {
            shouldRunGC = justRanFinalization;
            if (shouldRunGC) {
                justRanFinalization = false;
            } else {
                runGC = true;
            }
        }
        if (shouldRunGC) {
            Runtime.getRuntime().gc();
        }
    }


简单易懂啊，前面就是一些赋值啊判断啊，直接看最重要的一句

        if (shouldRunGC) {
            Runtime.getRuntime().gc();
        }
		
可以看到，只有当shouldRunGC为true时，才会真的去gc，而shouldRunGC的值其实不就是justRanFinalization的值吗，那好，我们全局搜索一下，发现justRanFinalization出现的地方取值可数,只有两个地方:

1、定义的地方

    /**
     * If we just ran finalization, we might want to do a GC to free the finalized objects.
     * This lets us do gc/runFinlization/gc sequences but prevents back to back System.gc().
     */
    private static boolean justRanFinalization;
2、赋值的地方

    /**
     * Provides a hint to the VM that it would be useful to attempt
     * to perform any outstanding object finalization.
     */
    public static void runFinalization() {
        boolean shouldRunGC;
        synchronized(lock) {
            shouldRunGC = runGC;
            runGC = false;
        }
        if (shouldRunGC) {
            Runtime.getRuntime().gc();
        }
        Runtime.getRuntime().runFinalization();
        synchronized(lock) {
            justRanFinalization = true;
        }
    }
	
哇咔咔，so easy，这时一个函数出现在我们眼前runFinalization(),这个函数是干嘛的？和System.gc()有什么关系，直接看函数的注释，其实就是执行对象的finalize()方法，但最关键的是，只有在这个地方是justRanFinalization的值才会赋为true，仔细观察，还有一个变量runGC，在System.gc()里会赋值为true，而在System.runFinalization()里只有runGC为
true时，才会真的去触发gc

##总结
所以说只是单纯的调用System.gc()，在Android5.0以上什么也不会发生，要么是
1. System.gc()和System.runFinalization()同时调用
2. 直接调用Runtime.getRuntime().gc()