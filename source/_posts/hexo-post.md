---
title: Hexo生成文章的时间问题
date: 2017-04-04 17:35:26
categories: hexo
tags: hexo
---

###问题描述
1. 文章以更新的时间排序而不是创建的事件

###问题分析
因为默认创建的文章没有date属相，导致每次更改文章，就会更新文章的date属相

###问题解决
####添加date属相
在``scaffolds``目录下的``post.md``中添加date属相，如下：
```
title: {{ title }}
categories:
tags:
date: {{ date }}
```
这样之后每次用`hexo new post `创建的文章，都会带上date属相，date的值为创建文章的当前事件，以后每次更新文章，只要不更改date的值，文章的事件就不会边

####生成错误
当上面改动完成，运行``hexo g``的时候发现会报错：
```
TypeError: Cannot read property 'offset' of null
    at Object.exports.timezone (E:\blog\qksblog\node_modules\hexo\lib\plugins\processor\common.js:43:40)
    at E:\blog\qksblog\node_modules\hexo\lib\plugins\processor\post.js:118:40
    at tryCatcher (E:\blog\qksblog\node_modules\hexo\node_modules\bluebird\js\main\util.js:26:23)
    at Promise._settlePromiseFromHandler (E:\blog\qksblog\node_modules\hexo\node_modules\bluebird\js\main\promise.js:505:31)
    at Promise._settlePromiseAt (E:\blog\qksblog\node_modules\hexo\node_modules\bluebird\js\main\promise.js:581:18)
    at Promise._settlePromises (E:\blog\qksblog\node_modules\hexo\node_modules\bluebird\js\main\promise.js:697:14)
    at Async._drainQueue (E:\blog\qksblog\node_modules\hexo\node_modules\bluebird\js\main\async.js:123:16)
    at Async._drainQueues (E:\blog\qksblog\node_modules\hexo\node_modules\bluebird\js\main\async.js:133:10)
    at Immediate.Async.drainQueues [as _onImmediate] (E:\blog\qksblog\node_modules\hexo\node_modules\bluebird\js\main\async.js:15:14)
    at processImmediate [as _immediateCallback] (timers.js:374:17)
```
原因是_confing.yml下的timezone配置错误：
需要将timezone配置成 时区名称：
```
timezone: Asia/Shanghai
```
