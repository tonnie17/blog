+++
title = "python logging模块的死锁问题"
date = 2018-05-01T22:15:31+08:00
tags = ["python"]
categories = [""]
draft = false
+++

某日，在排查线上问题时，在dump线程后发现了一些“诡异”的异常：

```shell
  File "/usr/local/lib/python3.5/logging/__init__.py", line 1838, in info
    root.info(msg, *args, **kwargs)
  File "/usr/local/lib/python3.5/logging/__init__.py", line 1271, in info
    Log 'msg % args' with severity 'INFO'.
  File "/usr/local/lib/python3.5/logging/__init__.py", line 1279, in info
    self._log(INFO, msg, args, **kwargs)
  File "/usr/local/lib/python3.5/logging/__init__.py", line 1415, in _log
    self.handle(record)
  File "/usr/local/lib/python3.5/logging/__init__.py", line 1425, in handle
    self.callHandlers(record)
  File "/usr/local/lib/python3.5/logging/__init__.py", line 1487, in callHandlers
    hdlr.handle(record)
  File "/usr/local/lib/python3.5/logging/__init__.py", line 853, in handle
    self.acquire()
  File "/usr/local/lib/python3.5/logging/__init__.py", line 804, in acquire
    self.lock.acquire()
```

异常的表现是logging模块在打日志的时候卡在了获取锁这个地方，并且一直处于阻塞的状态。众所周知，Python的logging模块是线程安全的，因为logging模块在输出日志时，都要获取一把锁，而这把锁是一把可重入锁，对于不同的线程，在打日志前都要等待这把锁变为可用状态，才能持有这把锁并执行提交逻辑，最典型的例子就像如下所示：

```python
def handle(self, record):
    """
    Conditionally emit the specified logging record.

    Emission depends on filters which may have been added to the handler.
    Wrap the actual emission of the record with acquisition/release of
    the I/O thread lock. Returns whether the filter passed the record for
    emission.
    """
    rv = self.filter(record)
    if rv:
        self.acquire()
        try:
            self.emit(record)
        finally:
            self.release()
    return rv
```

由上可知，如果说一个线程在获取锁时，另一个线程迟迟没有释放锁，那么这个线程就会一直阻塞在获取锁的这个步骤，但实际上这种情况基本不可能出现，因为输出日志是一个十分快的过程，不需要耗费太多时间。

因为环境是在多进程+多线程环境下运行，所以有理由怀疑问题与并发时的竞态有关，于是先写一个多进程的例子：

```python
import logging
import multiprocessing
import sys
from time import sleep

class CustomStreamHandler(logging.StreamHandler):
    def emit(self, record):
        sleep(0.1)
        super(CustomStreamHandler, self).emit(record)

root = logging.getLogger()
root.setLevel(logging.DEBUG)
root.addHandler(CustomStreamHandler(sys.stdout))

def g():
    logging.info(2)
    logging.info(2)

def f():
    logging.info(1)
    logging.info(1)

p = multiprocessing.Process(target=f)
p.start()
g()
```

主进程起了个新进程，调用f函数，打印日志“2”，接着并行调用g函数，打印日志“1”，为了看清楚问题，在logging handler获取锁后会sleep 0.1秒。

可以看到输出并没有问题：

```shell
1
2
1
2
```

现在做一些修改，把主线程的g函数的调用修改为在主进程新建一个线程来执行：

```python
import logging
import multiprocessing
import sys
from time import sleep

class CustomStreamHandler(logging.StreamHandler):
    def emit(self, record):
        sleep(0.1)
        super(CustomStreamHandler, self).emit(record)

root = logging.getLogger()
root.setLevel(logging.DEBUG)
root.addHandler(CustomStreamHandler(sys.stdout))

def g():
    logging.info(2)
    logging.info(2)

def f():
    logging.info(1)
    logging.info(1)

import threading
t = threading.Thread(target=g)
p = multiprocessing.Process(target=f)
t.start()
p.start()
```

执行输出，在程序输出两个2之后就立马阻塞住了：

```shell
2
2
阻塞...
```

为什么两个相似的例子中会出现不一样的结果呢？

仔细观察，在这个例子中，线程先于进程开始调用，那么假设线程先执行，换而言之这时候root logger的锁便被线程池持有了，按照逻辑，线程马上进入睡眠0.1秒，而进程这个时候也想要输出日志，它也想要获取这把锁，却一直获取不到，甚至直到线程退出了进程也一直在阻塞着。

这一切表象背后的实际原因，实际上可以从上面推敲出，当进程被fork时，线程实际上已经持出这把锁了，所以说子进程在复制地址空间后它认为这把锁还在占用状态，于是就一直等着，但它不知道这把锁永远不会被释放了...

而为什么在前一个例子中却没有任何异常呢？事实上，再根据之前说的，这把锁时一把可重入锁（RLock），它是属于线程级别的锁，即是这把锁在哪个线程acquire就一定要在哪个线程被release，再来看第一个例子，虽然主进程中起了一个子进程，但实际上这两个进程所处于的线程都是一样的（主线程），而第二个例子中，f和g函数是处于完全不同的线程当中，就是f想要释放掉这把锁，它也无能为力，因为这把锁由g创建，只能被g释放，并且由于线程相异，g没有办法把锁释放。

如果做一个投机取巧，把锁替换掉会怎样呢：

```python
import logging
import multiprocessing
import sys
from time import sleep
import threading

class CustomStreamHandler(logging.StreamHandler):
    def emit(self, record):
        sleep(0.1)
        super(CustomStreamHandler, self).emit(record)

root = logging.getLogger()
root.setLevel(logging.DEBUG)
root.addHandler(CustomStreamHandler(sys.stdout))

def g():
    print("g", threading.get_ident())
    handler = logging.getLogger().handlers[0]
    logging.info(2)
    logging.info(2)
    print(id(handler))
    print(id(handler.lock))

def f():
    print("f", threading.get_ident())
    handler = logging.getLogger().handlers[0]
    handler.createLock()
    logging.info(1)
    logging.info(1)
    print(id(handler))
    print(id(handler.lock))

import threading
print("main", threading.get_ident())
p = multiprocessing.Process(target=f)
t = threading.Thread(target=g)
t.start()
p.start()
```

执行这段代码，输出：

```python
main 140735977362240
g 123145405829120
f 140735977362240
2
1
2
1
4353914808
4353914808
4354039648
4349383520
```

会发现不会有任何阻塞了，因为在子进程中直接把这把状态为locked的锁替换掉了（一把全新的RLock），这时候输出handler和lock的地址，发现两个函数中的handler地址都是同一个，而lock已经是两个不一样的了（Copy-On-Write）。

## Solution

针对这个问题，Google很早之前已经给了一个解决方案：[python-atfork](https://github.com/google/python-atfork)

其主要思路是：对os.fork做一个monkeypatch，在fork调用时可触发三个自定义的hook：

```python
def atfork(prepare=None, parent=None, child=None):
```

- prepare：在fork子进程之前触发的函数。
- parent：在fork子进程之后父进程触发的函数。
- child：在fork子进程之后子进程触发的函数。

其中里面还给出了针对logging模块死锁问题的一个workaround：

```python
def fix_logging_module():
    logging = sys.modules.get('logging')
    # Prevent fixing multiple times as that would cause a deadlock.
    if logging and getattr(logging, 'fixed_for_atfork', None):
        return
    if logging:
        warnings.warn('logging module already imported before fixup.')
    import logging
    if logging.getLogger().handlers:
        # We could register each lock with atfork for these handlers but if
        # these exist, other loggers or not yet added handlers could as well.
        # Its safer to insist that this fix is applied before logging has been
        # configured.
        raise Error('logging handlers already registered.')

    logging._acquireLock()
    try:
        def fork_safe_createLock(self):
            self._orig_createLock()
            atfork.atfork(self.lock.acquire,
                          self.lock.release, self.lock.release)

        # Fix the logging.Handler lock (a major source of deadlocks).
        logging.Handler._orig_createLock = logging.Handler.createLock
        logging.Handler.createLock = fork_safe_createLock

        # Fix the module level lock.
        atfork.atfork(logging._acquireLock,
                      logging._releaseLock, logging._releaseLock)

        logging.fixed_for_atfork = True
    finally:
        logging._releaseLock()
```

确保了在fork子进程之后，锁是处于release状态的。

而在Python 3.7版本之后，官方也给出了标准库的方法实现相同的功能：

```shell
os.register_at_fork(*, before=None, after_in_parent=None, after_in_child=None)

Register callables to be executed when a new child process is forked using os.fork() or similar process cloning APIs. The parameters are optional and keyword-only. Each specifies a different call point.

 - before is a function called before forking a child process.
 - after_in_parent is a function called from the parent process after forking a child process.
 - after_in_child is a function called from the child process.
```

