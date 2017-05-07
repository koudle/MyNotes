---
title: GitHub 使用
date: 2017-05-07 21:03:25
categories: Git
tags:
---
## GitHub 使用

### 1.1 GitHub官方客户端
#### 下载 
 在[百度网盘](https://pan.baidu.com/s/1nv2WBDR)上下载并安装
### 安装
解压下载的压缩包，这是一个免安装的，不需要安装，只要在文件中找到github.exe，直接双击打开，为了以后使用方便，可以右键`发送到` -> `桌面快捷方式`
#### 登录
登录自己的GitHub帐号
#### 下载GitHub上的项目到本地(Clone)
点击右上角`+` -> `Clone`，选择自己的项目，然后选择`Clone`，之后选择要保存在本地的哪个路径
#### 修改
在本地打开自己的项目，这里推荐用[Sublime点击下载](http://pan.baidu.com/s/1o8hVWIA)。

Sublime使用方式:`File` -> `Open Folder`，打开本地的项目

然后选择一个文件，随便修改一下

#### GitHub界面介绍
当我们修改了代码之后，会希望把代码提交到GitHub上，如图

{%asset_img git.png GitHub%}

1. 是该项目
2. 该项目的提交记录，任意选中一个记录，可以在3中查看
3. 可以查看修改了哪些文件，以及文件中有哪些内容修改了。
4. 显示我们有没有还没提交的修改，当我们修改了内容之后，这里就会有提示
5. 点不同的圆圈会切换到不同的界面，可以自己试一下

#### 提交修改
点击上图的4，就进入到下图的界面:

{%asset_img commit.png GitHub%}

1. 当前修改的文件
2. 文件内容的变化
3. 这个是必填，是这次修改的注视，方便以后查看
4. 这个非必填
5. 最后点击这个提交

### Q&A

#### 1
`Q：Git与GitHub的区别`

`A:Git是一款免费、开源的分布式版本控制系统。而GitHub是用Git托管软件项目的一个网站平台。所以除了GitHub还有BitTorrent和GitLab，只是因为GitHub上托管了大量的开源项目，所以比较有名，但是Git绝不是GitHub，GitHub也绝不是Git。`