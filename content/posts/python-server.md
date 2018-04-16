+++
title = "Python服务器编程"
date = 2017-10-14T00:00:00+08:00
tags = ["python", "web"]
categories = [""]
draft = false
+++

IEEE公布的2017年编程语言排行榜，python高居首位。在百度指数上，python的搜索量也跻身到与java相等的量级，成为最火的语言之一。

![img](https://s1.ax2x.com/2018/04/13/NIvLQ.jpg)

那么Python适合用来做服务器编程吗？

![img](https://s1.ax2x.com/2018/04/13/NIy72.jpg)

首先，看看哪些公司在用Python作为服务器的主要技术栈？可以看到，其中不缺乏一些用户量庞大的公司。

![img](https://s1.ax2x.com/2018/04/13/NIaYa.jpg)

得益于语言的简洁性，python很适合用来进行快速开发，编写出可读性强的程序。那么怎么用python来做服务器编程呢？

从一个例子说起...

![img](https://s1.ax2x.com/2018/04/13/NIcPz.jpg)

这是一个简单的回显服务器，服务端每次从请求读取一些字节并返回给客户端。

![img](https://s1.ax2x.com/2018/04/13/NIr8S.jpg)

但由于服务器是单进程的，如果一个请求占住了服务器，就没办法处理另一个请求。

![img](https://s1.ax2x.com/2018/04/13/NIEHh.jpg)

这次做一些改动，每来一个请求就fork一个进程来处理，这样就不会出现之前的问题。

![img](https://s1.ax2x.com/2018/04/13/NIL4H.jpg)

但多进程模型处理不好会出现僵尸进程和孤儿进程，因此父进程需要处理SIGCHILD信号来收集退出的子进程的信息。

![img](https://s1.ax2x.com/2018/04/13/NIetN.jpg)

socketserver模块中ForkingMixIn收集子进程的例子：

![img](https://s1.ax2x.com/2018/04/13/NIK3u.jpg)

原始的CGI程序就是使用这种方式，对于每个请求都fork进程来解释cgi程序。

![img](https://s1.ax2x.com/2018/04/13/NI1M9.jpg)

不过随着请求数量的变多，fork进程所带来的开销往往很大。

![img](https://s1.ax2x.com/2018/04/13/NIALA.jpg)

所以CGI不仅慢...

![img](https://s1.ax2x.com/2018/04/13/NItRO.jpg)

而且

![img](https://s1.ax2x.com/2018/04/13/NIIYq.jpg)

甚至

![img](https://s1.ax2x.com/2018/04/13/NIkSd.jpg)

后来出现了FastCGI，它与CGI的区别，就是更Fast(误)，它是一个常驻进程，预先启动多个cgi进程来等待处理请求。

![img](https://s1.ax2x.com/2018/04/13/NI8HR.jpg)

不同于FastCGI，Apache搞了一套mod_python，使得python解释器可以嵌入在apache进程。

![img](https://s1.ax2x.com/2018/04/13/NIS4r.jpg)

后来PEP 333中定义了WSGI，成为沿用至今的Python web开发的标准协议。

![img](https://s1.ax2x.com/2018/04/13/NIf6Y.jpg)

应用WSGI协议的一个示例：

![img](https://s1.ax2x.com/2018/04/13/NIx3i.jpg)

绝大部分的python web开发框架都遵守了这套标准：

![img](https://s1.ax2x.com/2018/04/13/NI2My.jpg)

gunicorn是一个著名的wsgi http服务器，它采用pre-fork模型来处理和转发请求。（[原图出处](https://realpython.com/blog/python/kickstarting-flask-on-ubuntu-setup-and-deployment/)）

![img](https://s1.ax2x.com/2018/04/13/NI7eX.jpg)

![img](https://s1.ax2x.com/2018/04/13/NIRRl.jpg)

![img](https://s1.ax2x.com/2018/04/13/NIn0J.jpg)

gunicorn包含许多种worker模型：（[原图出处](https://www.spirulasystems.com/blog/2015/01/20/gunicorn-worker-types/)）

![img](https://s1.ax2x.com/2018/04/13/NIoaB.jpg)

![img](https://s1.ax2x.com/2018/04/13/NIqS6.jpg)

![img](https://s1.ax2x.com/2018/04/13/NN3Tp.jpg)

![img](https://s1.ax2x.com/2018/04/13/NN5J3.jpg)

抛开多进程，现在来看多线程的模型，该方案用线程代替进程来处理每一个请求：

![img](https://s1.ax2x.com/2018/04/13/NNB5G.jpg)

但是为什么许多人说python的多线程是个鸡肋呢？看下面同样的代码，用同步的方式和多线程的方式执行，多线程的代码却执行的更慢...

![img](https://s1.ax2x.com/2018/04/13/NNFMn.jpg)

这到底是什么回事？

![img](https://s1.ax2x.com/2018/04/13/NNHeE.jpg)

这就要说到python中的GIL了，由于GIL的制约，多线程很难充分利用cpu的性能（[原图引用](www.dabeaz.com/python/GIL.pdf)）

![img](https://s1.ax2x.com/2018/04/13/NNm02.jpg)

话虽如此，多线程在IO密集型应用上还是有不少用武之地的。下面是多线程在服务器编程的其中一些应用（[原图引用](http://www.brianstorti.com/the-actor-model/)）

Actor模型

![img](https://s1.ax2x.com/2018/04/13/NNzaa.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNVfz.jpg)

生产者－消费者

![img](https://s1.ax2x.com/2018/04/13/NN0TS.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNQJh.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNWIH.jpg)

concurrent.future在PEP 3148中被定义，它提供了更简单的多进程/多线程API

![img](https://s1.ax2x.com/2018/04/13/NNj5N.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNGOu.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNMU9.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNOgA.jpg)

在很长的一段时间，多进程/多线程的模型都应用的很好，但是

![img](https://s1.ax2x.com/2018/04/13/NNpQO.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNsaq.jpg)

这时候更适合服务器编程的IO多路复用模型开始被广泛应用：

![img](https://s1.ax2x.com/2018/04/13/NNufe.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNLiB.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNUX6.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNKZp.jpg)

![img](https://s1.ax2x.com/2018/04/13/NN1N3.jpg)

基于事件驱动的异步模型对服务器的资源的有效利用率显然易见（[原图出处](http://www.aosabook.org/en/twisted.html)）

![img](https://s1.ax2x.com/2018/04/13/NNt9K.jpg)

衍生了大量的异步网络框架

![img](https://s1.ax2x.com/2018/04/13/NN6bG.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNIUn.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNNnE.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNlWQ.jpg)

![img](https://s1.ax2x.com/2018/04/13/NN8c2.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNSia.jpg)

![img](https://s1.ax2x.com/2018/04/13/NNimz.jpg)

![img](https://s1.ax2x.com/2018/04/13/NkTDr.jpg)

![img](https://s1.ax2x.com/2018/04/13/NkmKi.jpg)

在Python 3.4后出现了专门处理异步IO的标准库asyncio

![img](https://s1.ax2x.com/2018/04/13/Nkzoy.jpg)

![img](https://s1.ax2x.com/2018/04/13/NkYdX.jpg)

![img](https://s1.ax2x.com/2018/04/13/Nk0rl.jpg)

![img](https://s1.ax2x.com/2018/04/13/NkQxJ.jpg)

而在随后的Python 3.5后出现了协程语法糖async/await

![img](https://s1.ax2x.com/2018/04/13/NkdzB.jpg)

![img](https://s1.ax2x.com/2018/04/13/Nkjh6.jpg)

虽然asyncio成为标准库，但它使用方法却较为复杂，不便于使用，也有人提议要asyncio提供更简洁的接口，也有不少的替代库出现

![img](https://s1.ax2x.com/2018/04/13/NkGkp.jpg)

![img](https://s1.ax2x.com/2018/04/13/NkOB3.jpg)

总的来说，服务器编程经历了从开始的简单到后来的复杂化最终慢慢演变到简单的方式上。

![img](https://s1.ax2x.com/2018/04/13/Nk6ld.jpg)

![img](https://s1.ax2x.com/2018/04/13/NkNBR.jpg)

