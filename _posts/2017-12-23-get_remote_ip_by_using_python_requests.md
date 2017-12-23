---
layout: post
title: requests get remote IP Address
category : 技术分享
tagline: "Supporting tagline"
tags : [编程细节]
published: true
---

# python requests get remote IP Address

## 引子
写爬虫的时候有个需求，获取到远端服务器的IP地址和端口。So easy，谷歌走起，于是得到了以下方式：

No.1

    rsp = requests.get(..., stream=True)
    rsp.raw._connection.sock.getpeername()
    
需要说明的是，我亲测了一下requests的2.14.2和2.18.4两个版本的python2和python3版本，其中，`connection`和`_connection`其实是一个对象实例。

    >>> rsp.raw.connection == rsp.raw._connection
    True

<!--break-->
    
其中`stream=True`是至关重要的。我们来看一下requests文档上是怎么写的：

> If you set stream to True when making a request, Requests cannot release the connection back to the pool unless you consume all the data or call Response.close. This can lead to inefficiency with connections. If you find yourself partially reading request bodies (or not reading them at all) while using stream=True, you should make the request within a with statement to ensure it’s always closed

大意就是说如果设置了这一项，我们将不会释放连接，直到你将连接中的数据取完或者主动关闭连接。注意这一点，下面我们还会说这个问题。

然后在接下来的使用过程中，我们发现一个令人崩溃的事实，这个方法时灵时不灵。不灵的时候的异常信息如下：

    >>> rsp.raw._connection.sock.getpeername()
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    AttributeError: 'NoneType' object has no attribute 'getpeername'
    
显而易见的是，`rsp.raw._connection`中的sock丢失了，是一个None。

这怎么办呢，经过本人仔细研究之后发现，导致不灵的原因在于http服务端主动关闭连接。为了验证这一点，我们自己搭建一个http服务，在响应客户端请求后，主动关闭连接（有兴趣可以看一下本人对Flask的研究报告）。同时对比http服务端保持连接（Keep-Alive）的情况，这种情况直接访问[百度](http://www.baidu.com)即可。

    >>> rsp = requests.get("http://127.0.0.1:12345", stream=True)
    >>> rsp.raw._connection.sock.getpeername()
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    AttributeError: 'NoneType' object has no attribute 'getpeername'
    >>> rsp.headers
    {'Date': 'Fri, 22 Dec 2017 23:19:01 GMT', 'Content-Length': '11', 'Content-Type': 'text/html; charset=utf-8', 'Server': 'Werkzeug/0.12.1 Python/2.7.10'}
    >>>
    >>> rsp = requests.get("http://www.baidu.com", stream=True)
    >>> rsp.raw._connection.sock.getpeername()
    ('111.206.223.206', 80)
    >>> rsp.headers
    {'Content-Encoding': 'gzip', 'Transfer-Encoding': 'chunked', 'Set-Cookie': 'BDORZ=27315; max-age=86400; domain=.baidu.com; path=/', 'Server': 'bfe/1.0.8.18', 'Last-Modified': 'Mon, 23 Jan 2017 13:27:32 GMT', 'Connection': 'Keep-Alive', 'Pragma': 'no-cache', 'Cache-Control': 'private, no-cache, no-store, proxy-revalidate, no-transform', 'Date': 'Fri, 22 Dec 2017 23:19:28 GMT', 'Content-Type': 'text/html'}

http响应头里的`Connection:Keep-Alive`表示服务器是长连接，等待客户端关闭。如果是`Connection:Close`或者没有这一项则表示服务器主动关闭了连接。

官方api文档上对`Response.close()`有一句描述：
> Releases the connection back to the pool. Once this method has been called the underlying raw object must not be accessed again.

大意是在连接关闭之后，raw对象将不能再被访问。结合以上对`stream`的一段引用，当stream中的内容取完之后，连接就会关闭，这意味着我们如果在获取服务器IP之前先获取了一下数据(无论何种方式，即使使用rsp.content)，以上的方法也是不可用的。

    >>> rsp = requests.get("http://www.baidu.com", stream=True)
    >>> data = rsp.content
    >>> rsp.raw._connection.sock.getpeername()
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    AttributeError: 'NoneType' object has no attribute 'sock'

这么多限制，这怎么玩？！有没有更通用一些方法呢？答案是有！研究之后，我们发现有以下两个方式可以在服务器主动关闭连接的情况下依然可用。

No.2

python2

    >>> rsp = requests.get("http://127.0.0.1:12345", stream=True)
    >>> rsp.raw._fp.fp._sock.getpeername()
    ('127.0.0.1', 12345)
    
python3

    >>> rsp = requests.get("http://127.0.0.1:12345", stream=True)
    >>> rsp.raw._fp.fp.raw._sock.getpeername()
    ('127.0.0.1', 12345)
  
No.3

python2

    >>> rsp = requests.get("http://127.0.0.1:12345", stream=True)
    >>> rsp.raw._original_response.fp._sock.getpeername()
    ('127.0.0.1', 12345)

python3

    >>> rsp = requests.get("http://127.0.0.1:12345", stream=True)
    >>> rsp.raw._original_response.fp.raw._sock.getpeername()
    ('127.0.0.1', 12345)
    
python3中需要加一个raw，此处不多说明，版本区别而已。另外需要注意的是，一定不要在主动关闭连接之后调用以上方法。

    >>> rsp.close()
    >>> rsp.raw._original_response.fp.raw._sock.getpeername()
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    AttributeError: 'NoneType' object has no attribute 'raw'
    


