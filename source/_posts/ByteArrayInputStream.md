---
title: Java I/O系列之InputStream
categories: Java
tags: Java
---
## 1.ByteArrayInputStream
ByteArrayInputStream是字节数组输入流，用于在内存中创建一个字节数组缓冲区，从输入流读取的数据保存在字节数组的缓冲区中。

### 1.1 结构
{% asset_img bytearrayinputstream.png ByteArrayInputStream结构图 %}

|方法|描述| 
|--|--|
|public int read()|从此输入流中读取下一个数据字节。|
|public int read(byte[] r, int off, int len)|将最多 len 个数据字节从此输入流读入字节数组。|
|public int available()|返回可不发生阻塞地从此输入流读取的字节数。| 
|public void mark(int read)|设置流中的当前标记位置。|
|public long skip(long n)|从此输入流中跳过 n 个输入字节。|
 
 注意：

* new一个ByteArrayInputStream的对象，需要一个byte数组作为缓冲区
* ByteArrayInputStream的close方法为空，所以关闭ByteArrayInputStream无效，因此此类的方法在关闭此流后仍可调用，而不会抛异常(IOException)
 
## 2.FileInputStream
FileInputStream是文件输入流，用于读取文件中的字节。

### 2.1结构
{% asset_img fileinputstream.png FileInputStream结构图 %}

|方法|描述| 
|--|--|
|public int read()|从输入流中读取下一个数据字节。|
|public int read(byte[] r, int off, int len)|将最多 len 个数据字节从此输入流读入字节数组。|
|public int available()|返回可不发生阻塞地从此输入流读取的字节数。| 
|public long skip(long n)|从此输入流中跳过 n 个输入字节。|
|public final FileDescriptor getFD() ||
|public FileChannel getChannel()||

## 3.FilterInputStream
FilterInputStream 的作用是用来“封装其它的输入流，并为它们提供额外的功能”。它的常用的子类有InflaterInputStream、BufferedInputStream、DataInputStream和PrintStream。

* BufferedInputStream的作用就是为“输入流提供缓冲功能，以及mark()和reset()功能”。

* DataInputStream 是用来装饰其它输入流，它“允许应用程序以与机器无关方式从底层输入流中读取基本 Java 数据类型”。

* PrintStream 是用来装饰其它输出流。它能为其他输出流添加了功能，使它们能够方便地打印各种数据值表示形式。

### 3.1 结构
{% asset_img filterinputstream.png FilterInputStream的结构图 %}

## 4.ObjectInputStream
ObjectInputStream用来序列化。ObjectInputStream 确保从流创建的图形中所有对象的类型与 Java 虚拟机中的显示的类相匹配。使用标准机制按需加载类。

ObjectOutputStream 和 ObjectInputStream 分别与 FileOutputStream 和 FileInputStream 一起使用时，可以为应用程序提供对对象图形的持久性存储。ObjectInputStream 用于恢复那些以前序列化的对象。其他用途包括使用套接字流在主机之间传递对象，或者用于编组和解组远程通信系统中的实参和形参。

ObjectOutputStream可以把对象直接存入到文件中,然后利用ObjectInputStream读取文件还原成对象,前提是该对象实现了Serializable接口.由于ObjectInputStream无法判断文件流中对象的数量,所以我们在循环读取的时候,只好写个死循环,然后捕捉EOFException异常,来实现把所有对象读进来.也可以在写入文件时,把所有对象存进ArrayList,然后把这个ArrayList写入文件,这样就不需要判断对象数量了.

```
需要实现 java.io.Serializable 或 java.io.Externalizable的接口 
```
## 5.PipedInputStream
在java中，PipedOutputStream和PipedInputStream分别是管道输出流和管道输入流。
它们的作用是让多线程可以通过管道进行线程间的通讯。在使用管道通信时，必须将PipedOutputStream和PipedInputStream配套使用。
使用管道通信时，大致的流程是：我们在线程A中向PipedOutputStream中写入数据，这些数据会自动的发送到与PipedOutputStream对应的PipedInputStream中，进而存储在PipedInputStream的缓冲中；此时，线程B通过读取PipedInputStream中的数据。就可以实现，线程A和线程B的通信。
