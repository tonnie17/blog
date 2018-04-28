+++
title = "网络笔记"
date = 2017-03-15T00:00:00+08:00
tags = ["web"]
categories = [""]
draft = false
+++

socket是一种ipc方法（介于传输层和应用层的一组api），允许同一主机或网络连接的主机上的应用程序交换数据。

现代操作系统支持下列socket:

- AF_UNIX: 允许同一主机的应用程序进行通信。
- AF_INET: 允许通过ipv4连接的主机的应用程序进行通信。
- AF_INET6: 允许通过ipv6连接的主机的应用程序进行通信。
- AF:地址族（address family）
- PF:协议族（protocol family）

![](https://s1.ax2x.com/2018/04/16/kLb23.png)

## socket类型

流(tcp)和数据报(udp)

![](https://s1.ax2x.com/2018/04/16/kLszK.png)

## 套接字选项

**SO_BROADCAST**：允许广播
**SO_KEPPALIVE**：周期性测试连接存活
**SO_LINGER**：若有数据报发送则延迟关闭
l_onoff=0：发送完发送缓冲区的数据并发送FIN。
l_onoff=1，l_linger=0：那么当close某个连接时TCP将终止该连接，丢弃发送缓冲区和接收缓冲区的数据并发送一个RST到对端，避免了TIME_WAIT。
l_onoff=1，l_linger!=0：发送完发送缓冲区的数据并发送FIN，丢弃接收缓冲区的数据，如果在CLOSED前延滞时间到，返回EWOULDBLOCK错误。
**SO_RCVBUF**：接收缓冲区大小
**SO_SNDBUF**：发送缓冲区大小
**SO_RCVLOWAT**：接收缓冲区低水平标记
**SO_SNDLOWAT**：发送缓冲区低水平标记
**SO_REUSEADDR**：重用地址

- 允许启动一个监听服务器并捆绑其众所周知的端口，即使以前建立的将端口用作它们的本地端口仍存在。
- 允许同一端口上启动同一服务器的多个实例，只要每个实例捆绑多个ip地址就行。
- 允许单个进程捆绑同一端口到多个套接字上。
- 允许完全重复的捆绑。

**TCP_NODELAY**：禁止Nagle算法
**TCP_MAXSEG**：TCP最大报文长度

**fnctl** 修改描述符性质。

## 套接字编程简介

新的通用套接字地址结构sockaddr_storage。

**字节操纵函数**

```
bzero: void bzero(void *dest, size_t nbyte)
```

把目标字节串指定书目的字节置为0。

```
memset: void memset(void *dest, int c, size_t len)
```

把目标字节串指定书目的字节置c。

**地址转换函数**

```
- inet_aton: int inet_aton(const char *strptr, in_addr *addrptr) // 点分十进制->32位网络子节串二进制
- inet_ntoa: char* inet_ntoa(struct in_addr addr) // 32位网络子节串二进制->点分十进制
```

```
- inet_pton: int inet_pton(int family, const char *strptr, void *addrptr) // 点分十进制->32位网络子节串二进制
- inet_
: char* inet_ntop(int family, const void *addrptr, char *strptr, size_t len) // 32位网络子节串二进制->点分十进制
```

**read, write**
字节流套接字调用read和write输入或输出的字节数可能比请求的字节数少，这个现象出现的原因是内核中缓冲区已达到极限。

**为什么诸如套接字地址结构长度的值-结果参数要用指针来传递?**

指针和指针长度的内容传递给内核，内核知道到底需要从进程复制多少数据进来。**原因**：当函数被调用时，结构大小是一个值，它告诉内核该结构大小，这样内核在写的时候不至于越界。当函数返回时，结构大小又是一个结果，它告诉进程内核在该地址结构存储了多少信息。这种类型的参数称为**值－结果**参数。(既是输入参数又是输出参数)

**为什么readn和writen函数都将void型指针转换成char型指针?**

在ANSI C标准中，不允许对void指针进行算术运算如pvoid++或pvoid+=1等，需要转换为char类型指针才能对指针进行加减操作。

## socket系统调用

- socket(),创建一个新socket。
- bind(),将socket绑定到一个地址上。
- listen(),允许一个流socket接受来自其他socket的连接。
- connect(),调用与另一个socket的连接。
- close(),关闭socket。

### 创建一个socket: socket()

```
int socket(int domain, int type, int protocol);
 - domain：与socket通信的domain。
 - type：socket类型，SOCK_STREAM（TCP），SOCK_DGRAM（UDP）。
 - protocol：通常指定为0，在RAW_SOCKET中为IPPROTO_RAW。
 return: 新创建socket的文件描述符。
```

### 将socket绑定到地址：bind()

```
int bind(int sockfd, const struct sockaddr* addr, socklen_t addrlen);
 - sockfd：在socket()调用取得的文件描述符。
 - addr：要socket绑定到的地址。
 - addrlen：制定了地址结构的大小。
 return：-1为绑定失败。
```

#### 通用socket地址结构struct sockaddr

```
struct sockaddr {
     sa_family_t sa_family;     //地址族
     char sa_data[14];     //socket地址
};
```

### 监听接入连接：listen()

```
int listen(int sockfd, int backlog);
 - sockfd：socket文件描述符。
 - backlog：限制未决连接的数量（在调用accept()前收到connect()的连接）。
return: -1为监听失败。
```

### 接受连接：accept()

```
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
 - sockfd：socket文件描述符。
 - addr：对端socket的地址结构。
 - addrlen：对端socket地址结构的长度。
return：和对端连接的文件描述符。
```

当调用accept()时，会创建一个新的socket，并且由这个新创建的socket来与执行connect()的对等socket进行连接。（这个socket并不绑定到新的端口号上，而是复制监听socket的地址和端口号，在tcp四元祖中记录主机和对端socket的地址信息）。

### 连接到对等socket：connect()

```
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
 - sockfd：socket文件描述符。
 - addr：要连接到socket的地址。
 - addrlen：地址结构的长度。
```

### 流socket i/o

![](https://s1.ax2x.com/2018/04/16/kLuvG.png)

一个socket可以使用close()系统调用来关闭或在应用程序终止后关闭，当对端读取完缓冲区的数据后，试图继续读取数据时会收到文件结束，试图写数据会收到SIGPIPE信号，系统调用产生EPIPE的错误。

#### 连接终止：close()

终止一个流socket的连接，如果多个文件描述符引用了一个socket，那么所有描述符被关闭后才会终止（若调用shutdown()则可以强制关闭socket上的信道）。

**socket上的标准系统调用**：
int read(fd, buf, bufsize);
int write(fd, buf, bufwrite);

**unix domain socket**
**unix domain socket地址：struct sockaddr_un**

```
struct sockaddr_un {
     sa_family_t sun_family;     //AF_UNIX
     char sun_path[108];     //socket路径名
};
```

当用来bind() UNIX DOMAIN SOCKET的时候，会在文件系统中创建一个条目（所在目录需要可读和可写）。

- 无法将socket绑定到既有路径名。
- socket和路径名为一对一关系。
- 当不再使用一个socket时使用unlink()或remove()删除条目。

权限：

- 要连接一个Unix Domain SOCKET需要在该socket文件上有写权限。
- 默认创建的socket赋予user,group,other权限，可用unmask()修改。

## 创建互联socket对：socketpair()

```
int socketpair(int domain, int type, int protocol, int sockfd[2]);
```

一个进程创建socket对，fork()出来的子进程复制socket对的文件描述符的副本，与父进程的socket队进行通信。

与手工创建socket对相比：不会绑定到任意地址上。

**Internet Domain Socket**

Socket的readline实现：

![](https://s1.ax2x.com/2018/04/16/kL4kn.png)

### Internet Domain Socket地址：

*ipv4*：struct sockaddr_in

```
struct in_addr {
     in_addr_t s_addr;     //32位无符号整数。
};
struct sockaddr_in {
     sa_family_t sin_family;     //地址族
     in_port_t sin_port;     //端口号
     struct in_addr sin_addr;     //ipv4地址
     unsigned char __pad[X];
};
```

### ip地址的格式转换：inet_pton()和inet_ntop()

### dns：

- 将主机名组织在一个层级的空间中。
- 一个节点的域名由该节点到根的路径所有节点组成的名字连接而成。
- 没有一个组织和系统管理整个层级。

![](https://s1.ax2x.com/2018/04/16/kLZBE.png)

名字与地址转换

**资源记录**
A：A记录把主机名映射为一个32位的IPV4地址。
CNAME：为常用的服务（FTP，WWW）指定CNAME记录。
静态主机文件：/etc/hosts

### getaddrinfo()：将主机名和服务名转换为ip名和端口号

```
int getaddrinfo(const char *host, const char *service, const struct addrinfo *hints, struct addrinfo *result);
 - host：主机名
 - service：服务名
 - hints：为如何选择getaddrinfo()返回的socket地址结构指定了更多的标准。
 - return: 0为成功，失败为非零值。
```

addrinfo细节：

```
struct addrinfo {
     int     ai_flags;
     int     ai_family;
     int     ai_protocol;
     int     ai_socktype;
     size_t     ai_addrlen;
     char     *ai_cannoname;
     struct sockaddr *ai_addr;
     struct addrinfo *ai_next;
}
```

释放addrinfo列表：freeaddrinfo()

```
void freeaddrinfo(struct addrinfo *result);
```

### getnameinfo()：给定一个socket地址结构，返回主机和服务名。

```
getnameinfo(const struct sockaddr *addr, socklen_t addrlen, char *host, size_t hostlen, char *service, size_t servlen, int flags);
 - addr：socket地址
 - addrlen：地址长度
 - host：主机名
 - hostlen：主机名长度
 - service：服务名
 - servlen：服务名长度
 - flags：其他参数
 - return: 0为成功，失败为非零值。
```

UNIX DOMAIN SOCKET与INET DOMAIN SOCKET比较：

- 一些实现上，UNIX DOMAIN SOCKET快于INET DOMAIN SOCKET。
- 可以使用目录权限来控制UNIX DOMAIN SOCKET的访问。
- UNIX DOMAIN SOCKET只针对在同一主机下应用程序下的网络通信。

#### 基本TCP套接字编程

**connect**

- 若TCP客户端没有收到SYN报文的响应，则返回ETIMEDOUT错误（等待超时）。
- 若客户端收到SYN的响应为RST，则表示服务器主机所指定的端口上没有相应的应用进程与主机连接，客户端一收到RST就马上返回ECONNREFUSED错误。
- 若客户端发出的SYN在中间路由器引发了一个destination unreachableICMP错误，则应把保存的信息作为EHOSTUNREACH或ENETUNREACH错误返回给进程。

**listen**

```
int listen(int sockfd, int backlog)
```

**内核为任何一个监听套接字维护两个队列**

1. 未完成连接队列（SYN_RCVD）。
2. 已完成连接队列（ESTABLISHED）。
   - listen的backlog参数曾被规定为两个队列总和的最大值。

当客户端的SYN到达时，若队列是满的，则忽略该分节（如果返回一个RST，则客户端无法判断“该端口没有服务器在监听”还是“该端口有服务器监听，只是它的队列满了”）。

**对一个TCP套接字调用close会导致发送一个FIN，随后是TCP连接终止序列，为什么父进程对connfd调用close没有终止客户端与它的连接呢？**
因为每个文件描述符或套接字都有一个**引用计数**，当父进程调用fork()时，connfd在父进程和子进程间共享，父进程和子进程的connfd的引用计数都为2，当父进程关闭connfd的连接时，此时引用计数为1，当子进程真正地处理和释放后，引用计数才为0。

**getsockname和getpeername**

- 在一个没有调用bind的客户端上，connect成功返回后，getsockname返回内核赋予该连接的本地ip地址和本地端口号。
- 在以端口号调用bind（告知内核去选择本地端口号）后，getsockname用于返回内核赋予的本地端口号。
- telnet调用getpeername过程：inetd父进程connfd = accept()->fork()->inetd子进程exec Telnet服务器->Telnet调用getpeername获取IP地址和端口号。

**正常终止客户端和服务器步骤**

1. 输入EOF后，fgets返回一个空指针，函数返回。
2. main调用exit终止。
3. 进程终止处理的部分工作是关闭所有打开的文件描述符，因此客户端打开的套接字由内核关闭，导致TCP发送一个FIN到服务器，服务器TCP回应ACK。至此，服务器套接字处于CLOSE_WAIT，客户端套接字处于FIN_WAIT2。
4. 当服务器接受FIN时，服务器子进程阻塞于readline调用，收到FIN的服务器端递送给子进程一个EOF，于是readline返回0。
5. 服务器子进程通过exit来终止。
6. 服务器子进程打开的所有描述符关闭，服务器发送FIN到客户端，客户端回应ACK，进入TIME_WAIT状态。
7. 子进程终止时，给父进程发送SIGCHLD信号，父进程处理子进程的关闭。

**服务器进程终止**

1. kill杀死服务器子进程，服务器向客户端发送FIN，客户端响应ACK。
2. SIGCHLD被处理。
3. 客户端阻塞在输入上。
4. 输入数据后，客户端把数据发送给服务器，服务器返回RST，客户端 调用writen后立即调用readline，由于收到FIN，readline读取EOF返回0，以“服务器过早终止”退出。

解决方案：select检测RST(两个EOF)。
**服务器关机**
init进程给所有进程发送SIGTERM信号，等待固定时间（5～20秒），发送SIGKILL信号（不可被捕获）。

**信号**
信号就是告知某个进程发生了某个事件的通知，有时也称为“软件中断”。信号可以：

- 进程->进程
- 内核->进程

处理僵死进程：waitpid

socket服务器

### 迭代型服务器：

每次只处理一个客户端，只有处理完后才能处理下一个客户端。

### 并发型服务器：

能处理多个客户端的请求。常见思路：

- 多进程：调用fork()为每一条连接新建一个子进程来处理，通过SIGCHLD信号来保证子进程不会变成僵尸进程。
- 多线程：为每一条连接新建一条线程来处理。
- 服务池（server pool）：预先创建好若干条进程/线程，放在一个集合中，当连接到来时，从池中取出服务程序处理。

### inetd：internet超级服务器守护进程

提供服务：

- 监视一组指定的套接字端口，按需启动其他服务（传递socket），降低系统运行的进程数量。
- 在/etc/inetd.conf中指定的每项服务，inetd都会创建一个恰当类型的套接字，绑定到指定的端口上，通过listen()调用运行客户端发来连接。实现了socket(),bind(),listen()的功能。

### 流式套接字的部分读和部分写：

readn()和writen()，循环启用系统调用，确保请求的字节数总是能够得到全部的传输。

#### shutdown

```
int shutdown(int sockfd, int how);
 - sockfd：socket的文件描述符。
 - how：关闭方式
   SHUT_RD：关闭连接的读端。
   SHUT_WR：关闭连接的写端。通过文件结尾告诉对端本地写端关闭。
   SHUT_RDWR：先执行SHUT_RD，后执行SHUT_WR。
```

与close()区别：有额外文件描述符引用的时候也会真正的关闭套接字。

### 套接字的系统调用：recv()和send()

```
ssize_t recv(int sockfd, void *buffer, size_t length, int flags);
ssize_t send(int sockfd, const void *buffer, size_t length, int flags);
部分flags参数：
 - MSG_DONTWAIT：非阻塞方式运行，如果没有数据可用，立即返回，错误码EAGAIN，通过fcntl()可以把套接字设置为非阻塞模式（O_NONBLOCK）。
 - MSG_WAITALL：阻塞方式运行。
 - MSG_NOSIGNAL：对端连接关闭时，不产生SIGPIPE信号。
```

### sendfile()系统调用

避免了内核空间的上下文切换，将文件内容直接传送到套接字上。

![](https://s1.ax2x.com/2018/04/16/kLcCS.png)

### 获取套接字地址getsockname()和getpeername()

```
getsockname(int sockfd, struct sockaddr *addr, socklen_t addrlen)：返回本地socket地址
getpeername(int sockfd, struct sockaddr *addr, socklen_t addrlen)：返回对端socket地址
```

### TCP状态迁移图

![](https://s1.ax2x.com/2018/04/16/kLhsQ.png)

### 监视套接字：netstat

### 监视tcp流量：tcpdump

### 套接字选项：setsockopt()

SO_REUSEADDR：当tcp端口释放时，无需等待TIME_WAIT即可直接重用地址。
SO_LINGER：close立即返回，但是当发送缓冲区中还有一部分数据的时候，系统将会尝试将数据发送给对端。SO_LINGER可以改变close的行为。

#### IO模型

**IO复用**：预先告知内核，使内核一旦发现进程指定的一个或多个IO条件就绪（输入准备被读取，或描述符能承接更多的输出），它就通知进程。
**输入操作的两个阶段**

- 等待数据准备好。
- 从内核向进程复制数据。

**阻塞IO模型**
recvfrom->无数据报准备好->等待数据->数据报准备好->数据从内核复制到用户空间->复制完成->返回成功指示
**非阻塞IO模型**
recvfrom->无数据报准备好->返回EWOULDBLOCK->recvfrom->无数据报准备好->返回EWOULDBLOCK->数据报准备好->数据从内核复制到用户空间->复制完成->返回成功指示
特点：轮询操作，大量占用cpu时间。
**IO复用模型**
select->无数据报准备好->据报准备好->返回可读条件->recvfrom->数据从内核复制到用户空间->复制完成->返回成功指示
**信号驱动模型**
建立信号处理程序(sigaction)->递交SIGIO->recvfrom->数据从内核复制到用户空间->复制完成->返回成功指示
**异步IO模型**
aio_read->无数据准备好->数据报准备好->数据从内核复制到用户空间->复制完成->递交aio_read中指定的信号
特点：直到数据复制完成产生信号的过程中进程都不被阻塞。

#### 描述符就绪条件

**读**

1. 套接字接收缓冲区的数据字节数大于等于套接字接收缓冲区低水平标记的当前大小（默认为1）。
2. 该连接的读半部关闭，读操作不阻塞且返回0。
3. 套接字是一个监听套接字且完成的连接数不为0。
4. 其上有一个套接字错误待处理，读操作不阻塞且返回-1。

**写**

1. 套接字发送缓冲区的数据字节数大于等于套接字接收缓冲区低水平标记的当前大小（默认为2048）。
2. 该连接的读半部关闭，对套接字的写操作产生SIGPIPE信号。
3. 使用非阻塞的connect套接字已建立连接，或者connect已失败告终。
4. 其上有一个套接字错误待处理（如RST），写操作不阻塞且返回-1。

**当某个套接字上发生错误时，它将select标记为可读且可写**

fd_set

```
FD_ZERO(&set); /*将set清零使集合中不含任何fd*/
FD_SET(fd, &set); /*将fd加入set集合*/
FD_CLR(fd, &set); /*将fd从set集合中清除*/
FD_ISSET(fd, &set); /*在调用select()函数后，用FD_ISSET来检测fd是否在set集合中，当检测到fd在set中则返回真，否则，返回假（0）*/
```

### IO多路复用

运行进程同时检查多个文件描述符以找出它们任意一个是否可以进行io操作，系统调用select()和poll()进行多路复用。

### 水平触发和边缘出发

EPOLLLT——水平触发
EPOLLET——边缘触发

- 水平触发：如果文件描述符可以非阻塞地进行io调用，此时认为他已经就绪）。（支持模型：select，poll，epoll）
- 边缘触发：如果文件描述符自上次来的时候有了新的io活动（新的输入），触发通知。（支持模型：信号驱动，epoll）

#### 边缘触发的饥饿问题

原因：文件描述符存在大量输入，一次次读取不完，由于边缘触发只在状态变化时进行通知，因此socket可能发生长时间的等待而导致饥饿。

解决方案：让应用程序维护一个列表，存放着已被标记为就绪态的文件描述符，通过一个循环的方式不断处理，直至出现EAGAIN或EWOULDBLOCK。

#### select()

```
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
 - readfds: 输入就绪的文件描述符集合。
 - writefds: 输出就绪的文件描述符集合。
 - exceptfds: 异常发生的文件描述符集合。
return: 就绪的文件描述符数量。
```

timeout参数

```
struct timeval {
    time_t    tv_sec;        //秒
    suseconds_t tv_usec;    //微秒
};
```

timeval参数：

- 0: 非阻塞
- NULL: 阻塞

#### poll()

```
int poll(struct pollfd fds[], nfds_t nfds, int timeout);
return: 就绪的文件描述符数量。
```

**pollfd结构**

```
struct pollfd {
    int fd;        //文件描述符
    short events;    //请求事件
    short revents;    //返回事件
};
```

![](https://s1.ax2x.com/2018/04/16/kLv12.png)

select()和poll()在套接字上通知的事件

![](https://s1.ax2x.com/2018/04/16/kLyqa.png)

#### select()和poll()的比较

- select()的fd_set对文件描述符的数量有上限（1024）。
- 在循环中调用select()时，要反复初始化fd_set。
- select()的超时精度比poll()高。
- 当文件描述符关闭时，select()返回-1，而poll()可获取被关闭的文件描述符。
- 当文件描述符的数量大时，select()和poll()的性能急剧下降。

### 信号驱动

当有输入数据来到指定的文件描述符时，内核向请求数据的进程发送一个信号，进程可以处理其他任务，通过接收信号以获得通知。

#### epoll()

在文件描述符上注册事件函数，由系统监视这些文件描述符，当在文件描述符可就绪时，内核通知应用进程。

##### 创建epoll实例epoll_create()

```
int epoll_create(int size);
 - size：制定了我们想要通过epoll检查的文件描述符个数。
```

##### 修改epoll的兴趣列表 epoll_ctl()

```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *ev);
 - epfd：要修改的文件描述符
 - op：执行的操作
 - fd：
     EPOLL_CTR_ADD：把fd添加到epfd的兴趣列表中。
     EPOLL_CTR_MOD：修改fd上设置的事件。
     EPOLL_CTR_DEL：将fd从epfd中移除。
 - ev: 指向结构体epoll_event的指针，设置关注事件。
```

epoll_event定义

```
struct epoll_event {
    uint32_t    events;        //epoll事件
    epoll_data_t    data;    //用户数据
};
```

##### 事件等待epoll_wait()

```
int epoll_wait(int epfd, struct epoll_event *evlist, int maxevents, int timeout);
 - epfd：文件描述符。
 - evlist：包含就绪文件描述符信息(epoll_event)的数组。
 - maxevents：最大事件数量。
 - timeout：超时设定，-1阻塞，0非阻塞，>0等待。
return: 就绪的文件描述符数量。
```

epoll与信号驱动

- 避免了信号处理的繁琐。
- 可以检查指定的事件类型（读就绪和写就绪）。
- 可以选择水平触发和边缘触发。

**Libevent**

对各种io模型（select,poll,signal,epoll）进行透明的封装，支持/dev/poll和kqueue接口。

### 高级IO函数

**recv,send**：允许通过第四个参数从内核到进程传递标志。
**readv,writev**：允许指定输入数据或输出数据的缓冲区变量。
**recvmsg,sendmsg**：具备发送和接收辅助数据的能力。

**套接字超时**

- 调用alarm和SIGALRM信号。
- 在select中阻塞io。
- 使用较新的SO_RCVTIMEO和SO_SNDTIMEO。

flags 说明 recv send
MSG_DONTROUTE 绕过路由表查找 y
MSG_DONTWAIT 仅本操作非等待 y y
MSG_OOB 发送或接收带外数据 y y
MSG_PEEK 窥看外来消息 y
MSG_WAITALL 等待所有数据（readn） y

**IO缓冲**

- 完全缓冲
- 行缓冲
- 不缓冲

**IO模型**

- select,poll：轮询，等待和数据复制都阻塞。
- epoll,kqueue：非轮询，事件驱动，内核到进程的数据复制阻塞。
- aio：非轮询，完全异步。

### 非阻塞IO

对应非阻塞的套接字：

- 输入操作(read,readv,recv,recvfrom,recvmsg)，缓冲区无数据则立即返回EWOULDBLOCK。
- 输出操作(write,writev,send,sento,sendmsg)，缓冲区无数据则立即返回EWOULDBLOCK。
- accept，无外来连接立即返回EWOULDBLOCK(当用select检测套接字时，总是把监听套接字设为非阻塞，防止select到accept间对端发送RST导致accept再度阻塞)。
- connect，连接不能立即建立，返回一个EINPROGRESS错误。

# TCP/IP

#### 理解面向连接和无连接协议之间的区别

**面向连接**和**无连接**指的都是协议，不是物理介质本身，而是说明如何在物理介质上传输数据的。
**无连接**：每个分组都是独立寻址，并由应用程序发送。
**面向连接**：协议实现维护了与后继分组有关的状态信息。
tcp：打电话。 udp：发邮件。

#### 理解子网和CIDR的概念

使用子网划分有助于防止路由表的增长，CIDR使得IP地址的分配更加有效，并使这些地址的层次化分配更加简单。

#### 理解私有地址和NAT

NAT：实现私有网络地址与全局网络地址间的映射。
NAT三种模式：

- 静态
- 地址池
- PAT

#### 套接字接口比XTI/TLI更好用

套接字提供了更简单、**可移植性**更好的接口。

#### 记住，TCP是一种流协议

TCP面向字节流，不存在**消息边界**，调用recv时，**不会对TCP发送给它的数据量做任何假设**。
**粘包**：由于TCP不存在消息边界，因此TCP发送的分组可能被对端一次性读取，造成粘包。

- 发送端需要等缓冲区满才发送出去，造成粘包
- 接收方不及时接收缓冲区的包，造成多个包接收

#### 不要低估TCP的性能

针对长时间的大数据的连接，TCP的性能会比UDP好得多。

#### 要认识到TCP是一个可靠的，但并不绝对可靠的协议

- 永久或临时的网络中断。
- 对等的应用程序崩溃。
- 运行对等应用程序的主机崩溃。

#### TCP/IP不是轮询的

TCP无法将连接的丢失立即通知应用程序。

检测死连接：

- keep-alive
- 心跳信号

#### 提防对等实体的不友好动作

- 检测客户端的终止
- 检测无效输入

#### 成功的LAN策略不一定能推广到WAN中去

WAN比LAN更容易出现网络时延问题。

#### 理解TCP的写操作

**写入操作**：把数据从用户缓冲区复制到内核，立即返回。不担保数据的正确发送。
写操作的错误时由读操作返回的，写操作只返回写调用时发生的明显错误。

- 文件描述符指向的不是套接字。
- 调用中指定的套接字不存在或未连接。
- 缓冲区地址参数指向无效地址。

#### 理解TCP的有序释放操作

通过shutdown来激活连接的有序释放，有序释放是在确保没有数据丢失的情况下拆除连接的一个过程。

#### 考虑用inetd来装载应用程序

inetd守护进程负责对连接或数据报进行监听，将套接字映射到stdin，stdout，stderr中。

wait|nowait：指明从inetd里头调用的服务是否可以自己处理socket。dgramsocket类型必须使用wait，而stream socket daemons，由于通常使用多线程方式，应当使用nowait。 wait 通常把多个 socket 丢给单个服务进程， 而 nowait 则 会为每个新的 socket 生成一个子进程。

#### 考虑使用两条TCP连接

派生出一个子进程来处理TTY连接的写操作，由父进程负责处理读操作。

#### 使应用程序成为事件驱动的

将多个定时器复用到一个select定时器中去，用这个函数支撑函数timeout和untimeout，只要少量工作就可以对多个事件进行定时。

#### 不要用TIME-WAIT暗杀来关闭一条连接

TIME-WAIT：拆除连接中发送最后一个ACK到连接关闭（CLOSED）的过程。
**TIME-WAIT作用**：维护连接状态，为耗尽网络中所有此连接的“走失段”提供时间，防止ACK丢失导致被动关闭的一端超时并重传FIN。如果此时连接关闭，TCP则会丢弃这条连接的记录，用RST（重置）来响应，对等实体会产生一个粗欧文状态，不会有序地终止。若此时端处于TIME-WAIT状态，则可以响应对端重传的FIN返回一个ACK。
提前终止TIME-WAIT：套接字的SO_LINGER选项。

#### 服务器应该设置SO_REUSEADDR选项

防止服务器等待前一条连接的TIME-WAIT状态过期而重启服务器时发生Address already in use错误，重启一个之前处于TIME-WAIT状态的服务器。

#### 可能的话，使用一个大规模的写操作，而不是多个小规模的写操作

Nagle算法：任意时刻，最多只能有一个未被确认的小段。
**Nagle算法作用**：防止在网络中传输大量的小报文导致网络泛洪。实现：在发送分组后，在发送剩余数据前等待ACK。
Nagle算法的规则： 　

1. 如果包长度达到MSS，则允许发送。
2. 如果该包含有FIN，则允许发送。
3. 设置了TCP_NODELAY选项，则允许发送。
4. 未设置TCP_CORK选项时，若所有发出去的小数据包（包长度小于MSS）均被确认，则允许发送。
5. 上述条件都未满足，但发生了超时（一般为200ms），则立即发送。

禁用Nagle算法：针对对延迟容忍度低的实时应用。

#### 理解如何使connect调用超时

- 使用alarm（告警）
- 使用select

#### 避免数据复制

- 进程使用共享缓冲区

#### 使用前将结构sockadddr_in清零

#### big endian和little endian

big endian：最高字节在地址最低位，最低字节在地址最高位，依次排列。
little endian：最低字节在最低位，最高字节在最高位，反序排列。

#### 不要将IP地址或端口号硬编入应用程序中
