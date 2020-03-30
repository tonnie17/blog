+++
title = "快速搭建一个Http2网站"
date = 2017-03-22T00:00:00+08:00
tags = ["http"]
categories = [""]
draft = false
+++

本文讲述了如何快速简单地搭建一个http2的网站。

要使用http2，你需要做三件事：

1. 搭建一个web server（nginx, apache, nodejs…)
2. 配置https
3. 开启http2

下面进入正题。

## 安装nginx

[Installing nginx](http://nginx.org/en/docs/install.html)

为了支持http2，nginx需要安装http_v2_module这个扩展，执行nginx -V来检查你的nginx是否已经安装了这个扩展：

```
nginx version: nginx/1.10.3
built by gcc 5.3.0 (Alpine 5.3.0)
built with OpenSSL 1.0.2k  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-http_ssl_module --with-http_realip_module --with-http_addition_module --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_stub_status_module --with-http_auth_request_module --with-http_xslt_module=dynamic --with-http_image_filter_module=dynamic --with-http_geoip_module=dynamic --with-http_perl_module=dynamic --with-threads --with-stream --with-stream_ssl_module --with-http_slice_module --with-mail --with-mail_ssl_module --with-file-aio --with-http_v2_module --with-ipv6
```

可以看到最后一行显示nginx已经安装了http2的扩展，nginx从1.9.5版本后就开始支持http_v2module这个扩展*，*如果你没有这个扩展，可以下载新的nginx源码包，编译时加入--with-http_v2_module这个参数：

```
./configure --prefix=/etc/nginx \
--sbin-path=/usr/sbin/nginx \
--modules-path=/usr/lib/nginx/modules \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/run/nginx.lock \
--http-client-body-temp-path=/var/cache/nginx/client_temp \
--http-proxy-temp-path=/var/cache/nginx/proxy_temp \
--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp \
--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp \
--http-scgi-temp-path=/var/cache/nginx/scgi_temp \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_realip_module \
--with-http_addition_module \
--with-http_sub_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-http_random_index_module \
--with-http_secure_link_module \
--with-http_stub_status_module \
--with-http_auth_request_module \
--with-http_xslt_module=dynamic \
--with-http_image_filter_module=dynamic \
--with-http_geoip_module=dynamic \
--with-http_perl_module=dynamic \
--with-threads \
--with-stream \
--with-stream_ssl_module \
--with-http_slice_module \
--with-mail \
--with-mail_ssl_module \
--with-file-aio \
--with-ipv6 \
--with-http_v2_module \
```

除此以外，由于ALPN逐渐取代NPN成为当前主流的http2协商协议，而OpenSSL 1.0.2 才开始支持 ALPN，要支持http2，你还需要把openssl的版本升级到1.0.2之上。

要升级openssl，你需要把openssl 1.0.2的源码包下载下来重新编译：

```
wget https://www.openssl.org/source/openssl-1.0.2-latest.tar.gz
```

执行openssl version查看openssl当前版本，已经为1.0.2了：

```
openssl version
OpenSSL 1.0.2k  26 Jan 2017
```

如果你对以上安装nginx和升级openssl的步骤感到麻烦，你也可以通过docker下载nginx alpine版本的镜像，里面已经内置了openssl的最新版本以及支持httpv2的nginx，只需要把证书和网站目录映射到nginx容器内部，运行docker容器：

```
docker run -d \
--name=nginx \
--net=host \
--privileged=true \
-v /etc/nginx:/etc/nginx \
-v /var/www:/var/www \
-v /etc/letsencrypt:/etc/letsencrypt \
nginx:stable-alpine
```

## 配置HTTPS

**Let's Encrypt**是一个提供免费SSL/TLS证书的数字证书认证机构，下面使用Let's Encrypt来为网站建立数字证书。

**certbot**是一个部署Let's Encrypt项目的自动化工具，打开[certbot](https://certbot.eff.org/)的官方页面，看到如下图的选择框：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-d085c5d17e66127ef1bbebef7e57fc03_1440w.png)

选择自己机器上使用的服务器（如Nginx，Apache...）以及操作系统，即可得到对应的操作步骤，例如我的是nginx+debian，官方给出了下面的安装和使用命令：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-e577b4e75f082472a6a056316eaa7098_1440w.png)

安装certbot：

```
sudo apt-get install certbot
```

接着使用certbot来部署证书，输入certbot certonly进入命令行的向导：

```
$ certbot certonly

How would you like to authenticate with the ACME CA?
-------------------------------------------------------------------------------
1: Spin up a temporary webserver (standalone)
2: Place files in webroot directory (webroot)
```

certbot提供了两种验证方式：

- **standalone**模式，certbot会在机器上建立一个临时的服务器来进行验证。
- **webroot**模式，这个模式需要你有一个运行的服务器，certbot会在服务器的隐藏路径创建一个文件来让Let's Encrypt服务端进行验证，以确定你对网站的拥有权。

官方推荐的是webroot模式，于是选择webroot模式。

在webroot模式下我们需要在web server配置一个路径以供Let's Encrypt进行验证，下面修改nginx的配置，目录/var/www/le即为Let's Encrypt进行验证的目录：

```
location ^~ /.well-known/acme-challenge/ {
  default_type "text/plain";
  root /var/www/le;
}

location = /.well-known/acme-challenge/ {
  return 404;
}
```

下一步输入你网站的域名：

```
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel):liangwentao.cc
```

然后下一步输入webroot的路径，就是之前nginx中root指令指向的目录：

```
Input the webroot for test.testnode.com: (Enter 'c' to cancel):/var/www/le
```

最后certbot验证成功，生成证书：

```
Waiting for verification...
Cleaning up challenges
Generating key (2048 bits): /etc/letsencrypt/keys/0000_key-certbot.pem
Creating CSR: /etc/letsencrypt/csr/0000_csr-certbot.pem
 
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/liangwentao.cc/fullchain.pem. Your cert
   will expire on 2017-06-18. To obtain a new or tweaked version of
   this certificate in the future, simply run certbot again. To
   non-interactively renew *all* of your certificates, run "certbot
   renew"
 - If you like Certbot, please consider supporting our work by:
 
   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
```

执行certbot certificates可以发现证书和私钥都已经在/etc/letsencrypt/live这个目录下了。

```
$ certbot certificates
Saving debug log to /var/log/letsencrypt/letsencrypt.log

-------------------------------------------------------------------------------
Found the following certs:
  Certificate Name: liangwentao.cc
    Domains: liangwentao.cc
    Expiry Date: 2017-06-18 04:09:00+00:00 (VALID: 87 days)
    Certificate Path: /etc/letsencrypt/live/liangwentao.cc/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/liangwentao.cc/privkey.pem
-------------------------------------------------------------------------------
```

最后还要在nginx上添加配置，使用刚才生成的证书：

```
server {
  listen 443 ssl;
  listen [::]:443 ssl ipv6only=on;
  server_name liangwentao.cc;
  ssl_certificate /etc/letsencrypt/live/liangwentao.cc/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/liangwentao.cc/privkey.pem;
  ssl_trusted_certificate /etc/letsencrypt/live/liangwentao.cc/chain.pem;
}
```

打开网页，发现左边有了把绿色的小锁了，说明https证书已经被正确安装了：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-fede2a570c2416e0119b4070213fca09_1440w.png)

对一个https网站进行抓包可以看到tls的通信过程：

1.

*Client Hello*

阶段，客户端向服务器提供支持的加密方法、压缩方法和协议版本，以及一个随机数，发送给服务器。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-4e8f13765ae89b304c86ff7a3316728a_1440w.png)

2.

*Server Hello*

阶段，服务器根据客户端传来的信息确定稍后使用的加密方法，压缩方法和协议版本，以及生成服务器端的随机数返回给客户端。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-1c3459afc0a07e8335ec86a513e3a274_1440w.png)

同时也把证书信息发送给客户端：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-c15641139ee919f72abde9c710222c8e_1440w.png)

必要的话服务器端会发送带有公钥信息的报文给客户端，让客户端得到服务器的证书和公钥，这个阶段叫**Server Key Exchange**。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-084748081df510112bb5efe2c97307fa_1440w.png)

3.紧接着客户端会对服务器端发送过来的证书进行验证，确认传输状态，这两个过程为

*Client Key Exchange*

和

*Change Cipher Spec*

阶段，确认无误后客户端生成一个随机数并用服务器的公钥加密，给服务器发送通知。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-33e9c3f017d9346c963b2e2a7754fc40_1440w.png)

4.最后服务器已经收到客户端的通知了，用私钥解密客户端发送过来的随机数，这时候双方都拥有3个随机数的信息了，于是可以通过加密方法生成对话密钥，加密发送的数据了。服务器这时候也进入

*Change Cipher Spec*

阶段，确认传输状态，双方可以进行加密的通信了，这时候服务器还会给客户端发送一个session ticket，客户端根据session ticket可以在客户端保存会话状态，这个session ticket是被密钥加密的，所以不用担心会话被窃听。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-5494cfdf7447b12f70a98f0a0cfea938_1440w.png)

## 启用http2

要启用http2，我们修改刚才nginx的配置，在监听的端口后面加上http2参数：

```
server {
  listen 443 http2 ssl;
  listen [::]:443 http2 ssl ipv6only=on;
  server_name liangwentao.cc;
  ssl_certificate /etc/letsencrypt/live/liangwentao.cc/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/liangwentao.cc/privkey.pem;
  ssl_trusted_certificate /etc/letsencrypt/live/liangwentao.cc/chain.pem;
}
```

重启nginx服务，打开网页，此时通过chrome的控制台可以看到协议一列的值为h2，说明网站使用http2协议进行通信：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-0ecf1d166a77f4967865d5b36fd2721d_1440w.png)

在chrome地址栏输入chrome://net-internals/#http2，可以看到当前http2的会话信息，点击session id，我们可以观察到这个会话的传输信息：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-a3e362ad9b7ee7875f61070e581321c6_1440w.png)

通过mimtproxy可以看出这个http2请求的一些信息，以下结果显示了这个请求使用了HTTP/2.0协议，TLS版本为1.2，证书机构为Let's Encrypt：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-3db1b9d02a23c535133bafd16d18513f_1440w.png)

至此，http2的升级就算完成了。

### ---

参考：

[专题 | JerryQu 的小站](https://imququ.com/series.html)

[SSL/TLS原理详解 - Sean's Notes - SegmentFault](https://segmentfault.com/a/1190000002554673)

