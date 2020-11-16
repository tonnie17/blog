+++
title = "Go调试方法"
date = 2019-12-13T00:00:00+08:00
tags = ["go"]
categories = [""]
draft = false
+++

## pprof

需要在服务添加监听的pprof端口：

```bash
import (
  "net/http"
	_ "net/http/pprof"
)

go func() {
  log.Println(http.ListenAndServe("localhost:"+os.Getenv("PORT"), nil))
}()
```

查看进程内存运行信息：

```bash
$ go tool pprof <http://localhost>:$PORT/debug/pprof/heap
```

查看30秒内cpu运行信息：

```bash
$ go tool pprof <http://localhost>:$PORT/debug/pprof/profile?seconds=30
```

查看goroutine阻塞信息：

```bash
$ go tool pprof <http://localhost>:$PORT/debug/pprof/block
```

查看5秒钟内的trace跟踪信息：

```bash
$ wget <http://localhost>:$PORT/debug/pprof/trace?seconds=5
```

查看mutex的holder信息：（需要在代码添加`runtime.SetMutexProfileFraction`）

```bash
$ go tool pprof <http://localhost:6060/debug/pprof/mutex>
```

### pprof输出说明

使用pprof profile时经常会输出以下信息：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/561f15b8-7d05-428e-9b93-916b9ac87a92/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/pprof.png)

其中这几列的含义：

- Flat：该函数自身执行的时长/内存
- Flat%：该函数自身执行的时长/内存占采样的百分比
- Cum：该函数自身+所有调用函数执行的时长/内存
- Cum%：该函数自身+所有调用函数执行的时长/内存占采样的百分比
- Sum%：代表从上到下Flat%的总和累计值

## 注意事项

1. **内存profiling记录的是堆内存分配的情况，以及调用栈信息**，并不是进程完整的内存情况，猜测这也是在go pprof中称为heap而不是memory的原因。
2. **栈内存的分配是在调用栈结束后会被释放的内存，所以并不在内存profile中**。
3. 内存profiling是基于抽样的，默认是每1000次堆内存分配，执行1次profile记录。
4. 因为内存profiling是基于抽样和它跟踪的是已分配的内存，而不是使用中的内存，（比如有些内存已经分配，看似使用，但实际以及不使用的内存，比如内存泄露的那部分），所以**不能使用内存profiling衡量程序总体的内存使用情况**。

## gdb

根据进程号查看系统调用（mac下使用dtruss）：

```bash
$ strace -f -p $PID
```

gdb调试（go build时候需要保留符号信息）：

```bash
$ gdb --pid=$PID $BINARY_FILE

(gdb) info goroutine //查看所有goroutine信息
(gdb) goroutine $GOROUTINE_ID bt //查看goroutine backtrace调用堆栈
(gdb) goroutine $GOROUTINE_ID frame //查看goroutine栈帧信息
```

gdb导出core dump文件：

```bash
(gdb) gcore
```

## dlv

使用[dlv](https://github.com/go-delve/delve)分析core文件：

```bash
$ dlv core $BINARY_FILE $CORE_FILE
```

启动dlv api服务：

```bash
$  dlv core $BINARY_FILE $CORE_FILE --listen :11111 --headless --log
```

获取goroutine 列表：

```bash
$echo -n '{"method":"RPCServer.ListGoroutines","params":[],"id":2}' | nc -w 1 localhost 11111 > list_goroutines.json
```

分类统计goroutine列表：

```bash
$ jq -c '.result[] | [.userCurrentLoc.function.name, .userCurrentLoc.line]' list_goroutines.json | sort | uniq -c
```

attach到运行的go进程：

```bash
$ dlv attach $PID
```

## 信号

```bash
A SIGQUIT, SIGILL, SIGTRAP, SIGABRT, SIGSTKFLT, SIGEMT, or SIGSYS signal causes the program to exit with a stack dump.
```

使进程退出并打印堆栈：

```bash
$ kill -SIGQUIT $PID
```
