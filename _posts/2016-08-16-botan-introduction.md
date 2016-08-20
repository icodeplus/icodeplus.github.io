---
layout: post
title:  "Botan库加解密用法总结"
date:   2016-08-16 21:10:00 +0800
categories:
- Botan
- QT
comments: true
---

> Botan库是c++开源加密解密算法库，支持多种加密解密算法，支持windows、mac、linux等多种平台。
Botan库支持不但支持编译成lib或dll加载使用，还支持生成单个.h和.cpp直接加入工程编译使用，直接包含源文件方法更加方便易用。

本文重点介绍在windows下采用源文件包含这种方式使用方法。

## 1、编译安装

#### 1、Botan官网https://botan.randombit.net/下载最新Botan库源文件（当前Botan-1.10.13）

#### 2、编译Botan库需要安装Python 2.7，下载安装Python

#### 3、解压源码zip压缩文件，命令行进入Botan-1.10.13目录，执行如下命令：
{% highlight shell %}
C:\Python27\python.exe configure.py --c=msvc --gen-amalgamation
{% endhighlight %}
命令执行完会在Botan-1.10.13目录下生成`botan_all.h`和`botan_all.cpp`两个文件

#### 4、将`botan_all.h`和`botan_all.cpp`两个文件导入使用的工程中即可编译使用

## 2、在QT中使用Botan库

把`botan_all.h`和`botan_all.cpp`两个文件导入使用工程，在`.pro`配置文件中加入如下配置：

{% highlight shell %}
# 编译时添加BOTAN_DLL空宏定义，取消dll import声明，否则编译错误
DEFINES += BOTAN_DLL=  

# 增加NOMINMAX宏定义，避免std::max/min 与windows.h中max/min冲突，NOMINMAX会取消windows.h中定义，否则编译错误
DEFINES += NOMINMAX

# Botan库使用了windows系统API，需要链接advapi32.dll和user32.dll两个库，否则链接失败
LIBS += -ladvapi32 -luser32
{% endhighlight %}

## 3、Botan库用法

* ### 管道/过滤器数据处理模式

Botan支持对流式数据处理，Filter包括压缩、加密和编码等多种处理，可以在Pipe上顺序安装多种过滤器，数据“流过”Pipe完成一些列处理动作，最终输出处理后的数据。
样例如下：

{% highlight c++ %}
Pipe pipe(new Base64_Encoder); // pipe owns the pointer, delete pointor
pipe.start_msg();
pipe.write("message 1");//消息编号0
pipe.end_msg(); // flushes buffers, increments message number

// process_msg(x) is start_msg() && write(x) && end_msg()
pipe.process_msg("message2");//消息编号1

std::string m1 = pipe.read_all_as_string(0); // 0号消息"message1"
std::string m2 = pipe.read_all_as_string(1); // 1号消息"message2"
{% endhighlight %}

Pipe负责安装Filter的删除，`Pipe pipe(new Base64_Encoder);` pipe析构释放`Base64_Encoder`对象。

多条消息处理，按编号读取处理结果

* ### AES加密样例

{% highlight c++ %}
AutoSeeded_RNG rng;//随机数生成器，可以满足99%的使用场景
SymmetricKey key(rng, 16); // a random 128-bit key
InitializationVector iv(rng, 16); // a random 128-bit IV

// The algorithm we want is specified by a string
Pipe pipe(get_cipher("AES-128/CBC", key, iv, ENCRYPTION));

pipe.process_msg("secrets");
pipe.process_msg("more secrets");

secure_vector<byte> c1 = pipe.read_all(0);

byte c2[4096] = { 0 };
size_t got_out = pipe.read(c2, sizeof(c2), 1);
// use c2[0...got_out]
{% endhighlight %}

* ### Botan库支持zip文件加密处理（需要安装zlib库支持）

{% highlight c++ %}
std::ifstream in("data.bin", std::ios::binary)
std::ofstream out("data.bin.bz2", std::ios::binary)

Pipe pipe(new Bzip_Compression);

pipe.start_msg();
in >> pipe;
pipe.end_msg();
out << pipe;
{% endhighlight %}

上述代码缺点是一次性读取文件内容到内存，如果遇到大文件则会耗费大量内存，处理性能较差，以下方法使用`DataSink_Stream`实现渐进式数据流处理，避免文件一次性加入内存

{% highlight c++ %}
std::ifstream in("data.bin", std::ios::binary)
std::ofstream out("data.bin.bz2", std::ios::binary)
//两个Filter：Bzip_Compression和DataSink_Stream
Pipe pipe(new Bzip_Compression, new DataSink_Stream(out));

pipe.start_msg();
in >> pipe;
pipe.end_msg();
{% endhighlight %}

* ### 支持Fork分支处理多种Filter

如下例子，同一message输入，按分支输出多个处理结果

{% highlight c++ %}
Pipe pipe(new Fork(
             new Fork(
                new Base64_Encoder,	//message 0
                new Fork(
                   0,		//message 1，空指针不对消息处理，原样返回前一Filter处理结果
                   new Base64_Encoder	//message 2
                   )
                ),
             new Hex_Encoder		//message 3
             )
   );
{% endhighlight %}

任何在`Fork`后加入`Pipe`的`Filter`都只处理`Fork`的第一个分支，对其他分支不处理

{% highlight c++ %}
Pipe pipe(new Fork(new Hash_Filter("SHA-256"),
                   new Hash_Filter("SHA-512")),
          new Hex_Encoder);
//上面等价于下面代码
Pipe pipe(new Hash_Filter("SHA-256"), new Hex_Encoder);
{% endhighlight %}

* ### Chain支持Filter合并

`Chain`将多种`Filter`连接在一起作为一个整体使用，可以混合`Fork`使用实现灵活分支处理

{% highlight c++ %}
Pipe pipe(new Fork(
              new Chain(new Hash_Filter("SHA-256"), new Hex_Encoder),
              new Hash_Filter("SHA-512")
              )
         );
{% endhighlight %}

`Chain`构造函数支持4个`Filter`参数，多于4个`Filter`通过传入数组构造

* ### 数据源

根据不同的使用场景，Botan库实现了4种数据源：

1. 内存数据源`DataSource_Memory`支持`byte`数组和字符串

2. 流数据源`DataSource_Stream`与STL I/O流兼容

3. 管道数据源`Pipe`，构造函数支持4个`Filter`参数，多于4个`Filter`通过传入数组构造

4. 队列数据源`SecureQueue`

`DataSink_Stream`作为一种`Filter`支持将数据输出到流中，可以实现数据流式处理，避免大数据加入内存

{% highlight c++ %}
DataSource_Stream in("in.txt");
Pipe pipe(get_cipher("AES-128/CTR-BE", key, iv),
          new DataSink_Stream("out.txt"));
pipe.process_msg(in);
{% endhighlight %}

* ### Filter分类

* 编码器

编码`Hex_Encoder`，`Base64_Encoder`

解码`Hex_Decoder`，`Base64_Decoder`

* Hashes and MACs

{% highlight c++ %}
Hash_Filter::Hash_Filter(std::string hash, size_t outlen = 0)
{% endhighlight %}

常用Hash有`MD5`、`SHA-1`、`Whirlpool`等

{% highlight c++ %}
MAC_Filter::MAC_Filter(std::string mac, SymmetricKey key, size_t outlen = 0)
{% endhighlight %}

* Cipher Filters

{% highlight c++ %}
Keyed_Filter *get_cipher(std::string cipher_spec, SymmetricKey key, InitializationVector iv, Cipher_Dir dir)
Keyed_Filter *get_cipher(std::string cipher_spec, SymmetricKey key, Cipher_Dir dir)
{% endhighlight %}

### 参考

[Botan: Crypto and TLS for C++]

[Botan: Crypto and TLS for C++]: https://botan.randombit.net/manual/