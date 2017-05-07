---
title: Hexo安装及建站
date: 2017-05-07 21:03:32
categories:
tags:
---
## Hexo安装及建站

### 1、node安装
#### 下载
[下载我](http://pan.baidu.com/s/1bpeU5YZ)
#### 安装
双击运行安装
#### 验证
打开cmd， `Windows 开始` -> `搜索程序和文件` -> `cmd` -> `cmd.exe` -> 打开

输入`node -v`

显示`v6.10.3`

输入`npm - v`

显示`3.10.10`

代表安装成功

#### 替换为淘宝的镜像
因为npm安装插件是从国外服务器下载，受网络影响大，可能出现异常，如果npm的服务器在中国就好了，所以这里要替换为淘宝的。

在cmd中运行`npm install cnpm -g --registry=https://registry.npm.taobao.org` 

这里会比较慢，休息一下在继续吧。

#### 验证
输入`cnpm -v`

显示
```
cnpm@4.5.0 (C:\Users\Administrator\AppData\Roaming\npm\node_modules\cnpm\parse_a
rgv.js)
npm@3.10.10 (C:\Users\Administrator\AppData\Roaming\npm\node_modules\cnpm\node_m
odules\npm\lib\npm.js)
node@6.10.3 (C:\Program Files\nodejs\node.exe)
npminstall@2.29.2 (C:\Users\Administrator\AppData\Roaming\npm\node_modules\cnpm\
node_modules\npminstall\lib\index.js)
prefix=C:\Users\Administrator\AppData\Roaming\npm
win32 x64 6.1.7601
registry=https://registry.npm.taobao.org
```
代表成功

### 2、安装hexo 
很简单，一行命令`npm install -g hexo-cli`

#### 验证hexo
输入`hexo -v`

显示
```
hexo: 3.2.2
hexo-cli: 1.0.2
os: Windows_NT 6.1.7601 win32 x64
http_parser: 2.7.0
node: 6.10.3
v8: 5.1.281.101
uv: 1.9.1
zlib: 1.2.11
ares: 1.10.1-DEV
icu: 58.2
modules: 48
openssl: 1.0.2k
```

代表hexo安装成功

下一步就是创建一个hexo的项目

### 3、建站
这里可以看[官网](https://hexo.io/zh-cn/docs/setup.html)，这里也说一下

#### 新建文件夹
在电脑本地新建一个文件夹，这里在桌面上新建一个叫hexo的文件夹

在cmd上输入`cd C:\Users\Administrator\Desktop\hexo`

`这里是先定位到目录，在init，和官网上先init，在定位到目录的效果一样`
#### init
输入`hexo init`

这里也比较慢。

运行完后用Sublime，打开hexo文件夹

#### 更换主题
在Sublime中可以看到一个theme的文件夹，打开，发现有一个叫landscape的文件夹，这个是hexo默认的主题，如果想要更改主题，把主题下载下来，放在theme的文件夹下

然后打开`_config.yml`

搜索ctrl+F`theme`

把`theme: landscape`中的landscape改成你的主题的文件夹的名字

#### 编译
为什么要编译，因为目前的文件，不能以网页的形式展现出来，所以要编译

输入`hexo g`

会生产一个public的文件，public里面的就是编译之后可以展示的文件。

#### 查看
输入 `hexo s`

然后在浏览器`http://localhost:4000`，就可以打开网页


#### 新建文章
`hexo new post ***`

***为文章的文件名
#### 注意
每一次修改都要先编译，才能看到变换，所以一般都是`hexo g`运行完后，在运行`hexo s`