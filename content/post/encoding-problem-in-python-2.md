---
title: "The Encoding Problem in Python 2"
date: 2018-08-06T23:15:46+08:00
draft: false
categories:
  - "开发"
tags:
  - "Python"
---

Python 2神坑之字符编码
==========




## 先来说一下Python 2 的编码坑

我们在写python代码时经常会遇到下边这两种报错：UnicodeDecodeError 和 UnicodeEncodeError：

**UnicodeDecodeError**: 'ascii' codec can't decode byte 0x87 in position 0: ordinal not in range(128)

**UnicodeEncodeError**: 'ascii' codec can't encode character u'\u5728' in position 1

这两种报错（异常）的产生都是因为Python 2 中的编码坑：

Python 2中有两种类型的字符串，str和unicode。

当执行一些操作，例如字符串合并（```str1 += str2```）、join函数（```"".join([str1, str2])```）时，如果操作的参数str1和str2一种是str型，而另一种是unicode型的话，Python 2 就会把操作的结果进行类型转化（通常是将结果统一成unicode类型）。而这个编码转化是Python自动帮你做的。举个例子：

```python
	In [6]: str1 = "fgetdapain"

	In [7]: str2 = u"is a nickname"

	In [8]: str1 + str2

	Out[8]: u'fgetdapainis a nickname'

	In [9]: "".join([str1, str2])

	Out[9]: u'fgetdapainis a nickname'

```



可以看到，str1是一个str型字符串，而str2是一个unicode型的字符串，在```+```和```join```两个操作后，操作的结果都变成了unicode类型。



然而，当str1或str2中有一些特殊字符的时候，例如str1和str2中的一个是从二进制文件中读取内容（例如从文件中读到一个字符：```'\x87'```，这个字符在Unicode字符集中就不存在），这时如果做转换就会出错：

```python
In [11]: str1 = "read from file: \x87"
In [12]: str2 = u"normal unicode string"
In [13]: str1 + str2
---------------------------------------------------------------
UnicodeDecodeError                        Traceback (most recent call last)
<ipython-input-13-01bd5b4ad288> in <module>()
----> 1 str1 + str2

UnicodeDecodeError: 'ascii' codec can't decode byte 0x87 in position 16: ordinal not in range(128)
```



然而，我们在写代码时，***这种字符串间的操作无处不在，稍有不慎，程序就回报出上述错误***。即使自己做好了统一字符串的类型，但是有可能用到了外部库，引用了库中的模块代码，还是有可能出现编码上的错误。



## 我最近遇到的编码错误场景

例如，我最近在用```httplib```这个Python标准库写一个自动POST文件到目标服务器的功能时，就遇到了编码问题。这里值得提一句，目标服务器接受的POST表单是multipart格式的。

在实现这个功能时，首先我按照服务器接受的参数和格式要求准备好了POST请求，并确保把它封装进字符串（这时我没有留意字符串的类型）。然后调用```httplib.HTTPConnection```类中的```request```方法去给服务器POST数据，而这个```request```方法的实现代码中，就有不少字符串操作，而这之中就会出现字符串类型的转换。

举个例子，下面这段代码是一段httplib的源码，调用```httplib.HTTPConnection```类的```request```方法就会调用此函数。```_send_output```函数的作用是将HTTP Header（Header存放在```self._buffer```）与HTTP Body拼装好，然后将整个请求发送出去。

```python
 def _send_output(self, message_body=None):
        """Send the currently buffered request and clear the buffer.

        Appends an extra \\r\\n to the buffer.
        A message_body may be specified, to be appended to the request.
        """
        self._buffer.extend(("", ""))
        msg = "\r\n".join(self._buffer)
        del self._buffer[:]
        # If msg and message_body are sent in a single send() call,
        # it will avoid performance problems caused by the interaction
        # between delayed ack and the Nagle algorithm.
        if isinstance(message_body, str):
            msg += message_body	#这里对msg做了+操作
            message_body = None
        self.send(msg)
        if message_body is not None:
            #message_body was not a string (i.e. it is a file) and
            #we must run the risk of Nagle
            self.send(message_body)
```

上边这个函数就很容易出现编码问题，假设我们传入的URL参数是一个unicode型字符串，例如是```url = u"http://blog.wangjiteng.club/somepath"```。用这个url拼装头部的时候，就会让整个头部字符串的类型是unicode型，然后，我们给HTTP Body传入一个str型的字符串，例如是向目标URL所POST的文件内容，文件中有一些二进制数据。

这时，在调用httplib发送请求时，执行到上边函数的```msg += message_body```一句，就回报错。这是由于头部信息```msg```是unicode型，而Body信息```message_body```是str型。执行这句代码时，会尝试将str型```message_body```转换为unicode型。但```message_body```中有二进制字节（例如```'\x87'```），unicode无法解析这种字节，就回抛出异常：

```python
UnicodeDecodeError: 'ascii' codec can't decode byte 0x87 in position 0: ordinal not in range(128)
```



## 如何避免

上面提到了，我在准备HTTP请求数据时，没有检查好所有数据字符串的类型，URL是unicode型，而请求体Body是str型，因此引发了这个编码错误。

**为了避免类似问题发生，我们在Python 2环境下使用字符串时，要尽量做到字符串类型的统一。或者对字符串做显示转换，避免上述隐式转换的发生。**

在写HTTP请求时，会处理很多用户传入的字符串，其中包括：url字符串，cookies类型，其他headers，body... 我们把这些字符串组合成HTTP请求，然后调用一些库，如: httplib, urllib, requests，来发送这些请求。

这些HTTP库的作者大多为老外，他们在写库的时候多用str类型来做字符串操作（例如httplib就是），因此在进行网络编程时，就要尽量避免编码问题。

总之，为了避免类似错误再次发生，我总结要注意以下几点：

1. 调用第三方的库时，要检查好传参时参数的字符串类型。确保类型统一，且与第三方库统一。
2. 在做网络编程、文件传送、解析二进制时，最好使用str类型来操作字符。因为如果用unicode会有一些二进制字符无法解码。
3. 自己写代码时，按照以下原则做好字符串类型的统一：
    - 做文本处理（如查找字符串中文字个数、切分字符串）时，统一用unicode型的字符串
    - 做 I/O处理（如，读写磁盘上的文件，打印一个字符串，网络通信等）时，使用str类型的字符串



