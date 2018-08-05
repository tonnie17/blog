+++
title = "Python Asyncio与多线程/多进程那些事"
date = 2018-06-27T16:33:17+08:00
tags = ["python"]
categories = [""]
draft = false
+++

根据asyncio的文档介绍，asyncio的事件循环不是线程安全的，一个event loop只能在一个线程内调度和执行任务，并且同一时间只有一个任务在运行，这可以在asyncio的源码中观察到，当程序调用`get_event_loop`获取event loop时，会从一个本地的Thread Local对象获取属于当前线程的event loop：

```
class _Local(threading.local):
    _loop = None
    _set_called = False

def get_event_loop(self):
    """Get the event loop.

    This may be None or an instance of EventLoop.
    """
    if (self._local._loop is None and
            not self._local._set_called and
            isinstance(threading.current_thread(), threading._MainThread)):
        self.set_event_loop(self.new_event_loop())

    if self._local._loop is None:
        raise RuntimeError('There is no current event loop in thread %r.'
                            % threading.current_thread().name)

    return self._local._loop
```

在主线程中，调用`get_event_loop`总能返回属于主线程的event loop对象，如果是处于非主线程中，还需要调用`set_event_loop`方法指定一个event loop对象，这样`get_event_loop`才会获取到被标记的event loop对象：

```
def set_event_loop(self, loop):
    """Set the event loop."""
    self._local._set_called = True
    assert loop is None or isinstance(loop, AbstractEventLoop)
    self._local._loop = loop
```

那么如果event loop在A线程中运行的话，B线程能使用它来调度任务吗？

答案是：不能。这可以在下面这个例子中被观察到：

```
import asyncio
import threading

def task():
    print("task")

def run_loop_inside_thread(loop):
    loop.run_forever()

loop = asyncio.get_event_loop()
threading.Thread(target=run_loop_inside_thread, args=(loop,)).start()
loop.call_soon(task)
```

主线程新建了一个event loop对象，接着这个event loop会在派生的一个线程中运行，这时候主线程想在event loop上调度一个工作函数，然而结果却是什么都没有输出。

为此，asyncio提供了一个`call_soon_threadsafe`的方法，专门解决针对线程安全的调用：

```
import asyncio
import threading

def task():
    print("task")

def run_loop_inside_thread(loop):
    loop.run_forever()

loop = asyncio.get_event_loop()
threading.Thread(target=run_loop_inside_thread, args=(loop,)).start()
loop.call_soon_threadsafe(task)
```

运行后可以看到，结果会输出task。

那么`call_soon_threadsafe`与`call_soon`相比，有什么区别呢？

其实他们之间的区别微乎其微，但`call_soon_threadsafe`与之相比主要在最后多了一个`_write_to_self`的调用：

```
def call_soon_threadsafe(self, callback, *args, context=None):
    """Like call_soon(), but thread-safe."""
    self._check_closed()
    if self._debug:
        self._check_callback(callback, 'call_soon_threadsafe')
    handle = self._call_soon(callback, args, context)
    if handle._source_traceback:
        del handle._source_traceback[-1]
    self._write_to_self()
    return handle
```

原来，event loop内部会维护着一个`self-pipe`，它由一对socketpair组成，`_write_to_self`的作用就是把一个信号写到`self-pipe`的一端，这样一来，event loop在检测到`self-pipe`发生事件后，就会响应并唤醒事件循环来处理任务，这时候事件循环就会去完成我们传入的回调。

类似`run_coroutine_threadsafe`内部也用到了`call_soon_threadsafe`。

在多线程下，虽然麻烦，但asyncio还是又办法应对的，而在多进程下，情况就更复杂了，[有的方法](https://link.zhihu.com/?target=https%3A//bugs.python.org/issue33688)甚至无法正确工作。

再举个例子来说，假设现在有一个场景，主进程运行着一个event loop，在某的时候会fork出一个子进程，子进程再去运行一个新建的event loop：

```
async def coro(loop):
    pid = os.fork()
    if pid:
        pass
    else:
        cloop = asyncio.new_event_loop()
        cloop.run_forever()

loop = asyncio.get_event_loop()
asyncio.ensure_future(coro(loop), loop=loop)
loop.run_forever()
loop.close()
```

表面上看起来没什么问题，父进程和子进程各自运行自己的event loop，可实际上在3.5及之前的版本中运行这段代码，会抛出一个让人摸不着头脑的错误：

```
...
cloop.run_forever()
  File "/Library/Frameworks/Python.framework/Versions/3.5/lib/python3.5/asyncio/base_events.py", line 411, in run_forever
    'Cannot run the event loop while another loop is running')
RuntimeError: Cannot run the event loop while another loop is running
```

大体说的是子进程在准备运行event loop时，检测到有其他的event loop正在运行。

必须观察的足够仔细，才能发现，这是asyncio内部设计所导致的问题，每当调用event loop的运行方法时，asyncio都会去检测正在运行的event loop，这同样是通过一个Thread Local对象来查找的：

```
class _RunningLoop(threading.local):
    _loop = None
_running_loop = _RunningLoop()
```

每次event loop开始运行后都会把里面的这个`_loop`变量赋值为当前的event loop，那么问题来了，当发生fork调用时，这个数据结构也会被复制到子进程，这时候子进程便会因为检测到有正在运行的event loop，而中断并且抛出异常。

所幸的是，在3.6版本过后，这个问题已经被修复，解决方法也很简单，即为`_RunningLoop`新增一个pid属性，用来标识event loop所属的进程，在获取正在运行的event loop时同时检测进程号是否相等：

```
def _get_running_loop():
    """Return the running event loop or None.

    This is a low-level function intended to be used by event loops.
    This function is thread-specific.
    """
    # NOTE: this function is implemented in C (see _asynciomodule.c)
    running_loop, pid = _running_loop.loop_pid
    if running_loop is not None and pid == os.getpid():
        return running_loop
```

这只是asyncio在多进程影响下的其一个缩影，话说回来，在asyncio的实现中大部分都没有采取类似这种进程追踪的方式或者at_fork钩子来检测fork事件的发生。我们知道，asyncio的事件循环内部维护了selector以及其监听的文件描述符（fd），当发生fork调用时，这些数据也会一并复制到子进程中，这就可能导致一个问题，就是当fd变为就绪时，父进程跟子进程有可能同时受到影响，程序便会出现混乱。

于是有人提议说当fork后子进程需要立即调用`loop.close()`关闭事件循环，否则当它使用这个event loop时就抛出`RuntimeError`异常，但这样又有可能导致潜在的混乱，因为调用`loop.close()`会从底层删除监听的描述符，而这个操作也会顺带造成父进程的影响；所以又有人提议说，在子进程调用`loop.close()`后只关闭selector本身，而不去对底层的fd进行操作，同时重置default event loop，这样一来子进程既能保证父进程的event loop不受影响的同时也能保证自身的可靠。但毕竟这都只是提议，asyncio要真正支持`os.fork`目前还没有提上日程（在3.8之前）。

------

对此感兴趣的朋友可以关注以下的两个issue：

[Issue 21998: asyncio: support fork](https://link.zhihu.com/?target=https%3A//bugs.python.org/issue21998)

[Issue 22087: asyncio: support multiprocessing (support fork)](https://link.zhihu.com/?target=https%3A//bugs.python.org/issue22087)
