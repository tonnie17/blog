+++
title = "Python服务器编程"
date = 2017-10-14T00:00:00+08:00
tags = ["python", "web"]
categories = [""]
draft = false
+++

IEEE公布的2017年编程语言排行榜，python高居首位。在百度指数上，python的搜索量也跻身到与java相等的量级，成为最火的语言之一。

![img](https://pic3.zhimg.com/80/v2-68b22b88b69c321ca3975cce29c2e1a6_hd.jpg)

那么Python适合用来做服务器编程吗？

![img](https://pic1.zhimg.com/80/v2-f2ba2ad2d58be5f2b896ba5a6239b700_hd.jpg)

首先，看看哪些公司在用Python作为服务器的主要技术栈？可以看到，其中不缺乏一些用户量庞大的公司。

![img](https://pic4.zhimg.com/80/v2-96ff84eeecabd87546415bca617b2303_hd.jpg)

得益于语言的简洁性，python很适合用来进行快速开发，编写出可读性强的程序。那么怎么用python来做服务器编程呢？

从一个例子说起...

![img](https://pic3.zhimg.com/80/v2-14d268c6627307ee0f4764565ab3f6ae_hd.jpg)

这是一个简单的回显服务器，服务端每次从请求读取一些字节并返回给客户端。

![img](https://pic3.zhimg.com/80/v2-1486ef5191b3ee4d6a2fe16b67908872_hd.jpg)

但由于服务器是单进程的，如果一个请求占住了服务器，就没办法处理另一个请求。

![img](https://pic4.zhimg.com/80/v2-9678e3c91060886fff4c4676ad6e72bb_hd.jpg)

这次做一些改动，每来一个请求就fork一个进程来处理，这样就不会出现之前的问题。

![img](https://pic4.zhimg.com/80/v2-534ea6022aba9039a7a2189d67d6eb23_hd.jpg)

但多进程模型处理不好会出现僵尸进程和孤儿进程，因此父进程需要处理SIGCHILD信号来收集退出的子进程的信息。

![img](https://pic4.zhimg.com/80/v2-f95b7f9931fea4725288c36b898036e7_hd.jpg)

socketserver模块中ForkingMixIn收集子进程的例子：

![img](https://pic2.zhimg.com/80/v2-a5d2dccbdf7fffe92ea6cbd599eb7919_hd.jpg)

原始的CGI程序就是使用这种方式，对于每个请求都fork进程来解释cgi程序。

![img](https://pic2.zhimg.com/80/v2-7027dd8c1729f7800a4d5e2a5d0528f5_hd.jpg)

不过随着请求数量的变多，fork进程所带来的开销往往很大。

![img](https://pic4.zhimg.com/80/v2-486d9019afe232ba122f2d26bf910523_hd.jpg)

所以CGI不仅慢...

![img](https://pic3.zhimg.com/80/v2-9ebcad8d16db10126e51fef24236ddd6_hd.jpg)

而且

![img](https://pic1.zhimg.com/80/v2-f6b2cafb3188eb382ed47573e2895a74_hd.jpg)

甚至

![img](https://pic4.zhimg.com/80/v2-7ed8519f9a5d4f28502aab876e67492f_hd.jpg)

后来出现了FastCGI，它与CGI的区别，就是更Fast(误)，它是一个常驻进程，预先启动多个cgi进程来等待处理请求。

![img](https://pic3.zhimg.com/80/v2-90ace8dab1832ce16ac85823070239c6_hd.jpg)

不同于FastCGI，Apache搞了一套mod_python，使得python解释器可以嵌入在apache进程。

![img](https://pic2.zhimg.com/80/v2-4fe6b63dd9ab9971cc68067a71b7f159_hd.jpg)

后来PEP 333中定义了WSGI，成为沿用至今的Python web开发的标准协议。

![img](https://pic3.zhimg.com/80/v2-fe9158a4418ac34bc0b3bd94b130626e_hd.jpg)

应用WSGI协议的一个示例：

![img](https://pic1.zhimg.com/80/v2-86c890d195e5b2e5844eeac1c2e7c074_hd.jpg)

绝大部分的python web开发框架都遵守了这套标准：

![img](https://pic4.zhimg.com/80/v2-0ad754a1c61296c21b64382d870435f7_hd.jpg)

gunicorn是一个著名的wsgi http服务器，它采用pre-fork模型来处理和转发请求。（[原图出处](https://realpython.com/blog/python/kickstarting-flask-on-ubuntu-setup-and-deployment/)）

![img](https://pic3.zhimg.com/80/v2-a9b9f36bcc04c2d94d634ecc75abf776_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-447bf828eb774e4c12a3a5c3715bc3ac_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-329c02e66b84cda5a543726315ba9a18_hd.jpg)

gunicorn包含许多种worker模型：（[原图出处](https://www.spirulasystems.com/blog/2015/01/20/gunicorn-worker-types/)）

![img](https://pic1.zhimg.com/80/v2-713748e2c071f5e4cff8b4020238de04_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-fe73f2e2e721b696765ca58089a0ab1c_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-09fa19d529defa4177c8d093bab76842_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-db8830e2cc4d58f7e1015ea0a04af51a_hd.jpg)

抛开多进程，现在来看多线程的模型，该方案用线程代替进程来处理每一个请求：

![img](https://pic4.zhimg.com/80/v2-20b74f2524219ea28c45ee64952e6acb_hd.jpg)

但是为什么许多人说python的多线程是个鸡肋呢？看下面同样的代码，用同步的方式和多线程的方式执行，多线程的代码却执行的更慢...

![img](https://pic4.zhimg.com/80/v2-41a84e900c6842a603570bf5233e751f_hd.jpg)

这到底是什么回事？

![img](https://pic2.zhimg.com/80/v2-3a7b385f831156a22b871bdf5fc9d755_hd.jpg)

这就要说到python中的GIL了，由于GIL的制约，多线程很难充分利用cpu的性能（[原图引用](www.dabeaz.com/python/GIL.pdf)）

![img](https://pic3.zhimg.com/80/v2-5124b653582da0b877a47893c9ebbe3a_hd.jpg)

话虽如此，多线程在IO密集型应用上还是有不少用武之地的。下面是多线程在服务器编程的其中一些应用（[原图引用](http://www.brianstorti.com/the-actor-model/)）

Actor模型

![img](https://pic2.zhimg.com/80/v2-251c5cca31d0b3abecb6e6391427448d_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-04822f768100441595d6491835c12c88_hd.jpg)

生产者－消费者

![img](https://pic3.zhimg.com/80/v2-c7ccdb3ec379a3d1b3e57819f61745a2_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-e8d736eff5c90c2494283866d87dcdd6_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-bdceaec322c7dfefc65199e26f9d9ead_hd.jpg)

concurrent.future在PEP 3148中被定义，它提供了更简单的多进程/多线程API

![img](https://pic2.zhimg.com/80/v2-9b754f1a252b4fbf408a2bf4b18c0ce5_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-5701797b9b21ae9bcde9f5fa0c1b1d32_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-cd8d5c4d035658188db5c359fe1b4212_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-af56d7404e1c86386f59ca7e7a5b174b_hd.jpg)

在很长的一段时间，多进程/多线程的模型都应用的很好，但是

![img](https://pic3.zhimg.com/80/v2-96c6752abf7de6a85212400c5021440a_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-4cf13015cb400b478bf45a2eba616a68_hd.jpg)

这时候更适合服务器编程的IO多路复用模型开始被广泛应用：

![img](https://pic1.zhimg.com/80/v2-c0eb79beeb44652260d71d09d18133f4_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-22a96b790ee2e4f0f4efeae772323d0f_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-238d79e141c9750b4ad12fe2cd1ed4c7_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-2270adf93636d95323da58add1191aa7_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-6fdd6eb4b35ca32a310d775fca57e995_hd.jpg)

基于事件驱动的异步模型对服务器的资源的有效利用率显然易见（[原图出处](http://www.aosabook.org/en/twisted.html)）

![img](https://pic1.zhimg.com/80/v2-3705b896482459f2fb5854668941d81c_hd.jpg)

衍生了大量的异步网络框架

![img](https://pic4.zhimg.com/80/v2-e1e15ce84f0aade75c9b9624bd48388b_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-9627643585f5d71507498fd889542342_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-44a26e9d1a5b9c6e16ee061950ba7265_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-1d2bc2ff12ec26d8b45825862f2d52f1_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-9c51c194e68ceeb897ab850c0cdc9d4e_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-d12b42d45630cffefd2a9ec12de1efe3_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-9b743a50522d4d8e042538c56ca712b9_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-14842f71acb0c608104f173d72b121fc_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-9c95d7dc8b15165f366ac6dcbfe39637_hd.jpg)

在Python 3.4后出现了专门处理异步IO的标准库asyncio

![img](https://pic4.zhimg.com/80/v2-3dcf8ba7e3fa0a5920de529aa43d331b_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-710572e3c5a4f4a135de1b90a32cea98_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-cd94267453d66a9a21c877e442d8b576_hd.jpg)

![img](https://pic3.zhimg.com/80/v2-0e9277a69f45eef11c5ec7f693cfe0c6_hd.jpg)

而在随后的Python 3.5后出现了协程语法糖async/await

![img](https://pic3.zhimg.com/80/v2-7bf65c94270b7cc26fd88b0b826120fe_hd.jpg)

![img](https://pic4.zhimg.com/80/v2-22fcaba5274c452dcf485aab8b20c5fb_hd.jpg)

虽然asyncio成为标准库，但它使用方法却较为复杂，不便于使用，也有人提议要asyncio提供更简洁的接口，也有不少的替代库出现

![img](https://pic2.zhimg.com/80/v2-89a30270641104fde58d6928cd68c8c9_hd.jpg)

![img](https://pic1.zhimg.com/80/v2-517239a26134987a066292e448b67424_hd.jpg)

总的来说，服务器编程经历了从开始的简单到后来的复杂化最终慢慢演变到简单的方式上。

![img](https://pic4.zhimg.com/80/v2-b3b3a842307f5d88e84faba1c72162cf_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-d42923e92168d4b6f36870747413c9c5_hd.jpg)

