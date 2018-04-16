+++
title = "Python Async/Await入门指南"
date = 2017-06-05T00:00:00+08:00
tags = ["python"]
categories = [""]
draft = false
+++

本文将会讲述Python 3.5之后出现的async/await的使用方法，以及它们的一些使用目的，如果错误，欢迎指正。

昨天看到David Beazley在16年的一个演讲：[Fear and Awaiting in Async](https://www.youtube.com/watch%3Fv%3DE-1Y4kSsAFc)，给了我不少的感悟和启发，于是想梳理下自己的思路，所以有了以下这篇文章。

Python在3.5版本中引入了关于协程的语法糖async和await，关于协程的概念可以先看我在[上一篇文章](http://4fish.xyz/python-async)提到的内容。

看下Python中常见的几种函数形式：

  1.普通函数

```python
def function():
    return 1
```

  2.生成器函数

```python
def generator():
    yield 1
```

在3.5过后，我们可以使用async修饰将普通函数和生成器函数包装成异步函数和异步生成器。

  3.异步函数（协程）

```python
async def async_function():
    return 1
```

  4.异步生成器

```python
async def async_generator():
    yield 1
```

通过类型判断可以验证函数的类型

```python
import types
print(type(function) is types.FunctionType)
print(type(generator()) is types.GeneratorType)
print(type(async_function()) is types.CoroutineType)
print(type(async_generator()) is types.AsyncGeneratorType)
```

直接调用异步函数不会返回结果，而是返回一个coroutine对象：

```python
print(async_function())
# <coroutine object async_function at 0x102ff67d8>
```

协程需要通过其他方式来驱动，因此可以使用这个协程对象的send方法给协程发送一个值：

```python
print(async_function().send(None))
```

不幸的是，如果通过上面的调用会抛出一个异常：

```python
StopIteration: 1
```

因为生成器/协程在正常返回退出时会抛出一个StopIteration异常，而原来的返回值会存放在StopIteration对象的value属性中，通过以下捕获可以获取协程真正的返回值：

```python
try:
    async_function().send(None)
except StopIteration as e:
    print(e.value)
# 1
```

通过上面的方式来新建一个run函数来驱动协程函数：

```python
def run(coroutine):
    try:
        coroutine.send(None)
    except StopIteration as e:
        return e.value
```

在协程函数中，可以通过await语法来挂起自身的协程，并等待另一个协程完成直到返回结果：

```python
async def async_function():
    return 1

async def await_coroutine():
    result = await async_function()
    print(result)
    
run(await_coroutine())
# 1
```

要注意的是，await语法只能出现在通过async修饰的函数中，否则会报SyntaxError错误。

而且await后面的对象需要是一个Awaitable，或者实现了相关的协议。

查看Awaitable抽象类的代码，表明了只要一个类实现了`__await__`方法，那么通过它构造出来的实例就是一个Awaitable：

```python
class Awaitable(metaclass=ABCMeta):
    __slots__ = ()

    @abstractmethod
    def __await__(self):
        yield

    @classmethod
    def __subclasshook__(cls, C):
        if cls is Awaitable:
            return _check_methods(C, "__await__")
        return NotImplemented
```

而且可以看到，Coroutine类也继承了Awaitable，而且实现了send，throw和close方法。所以await一个调用异步函数返回的协程对象是合法的。

```python
class Coroutine(Awaitable):
    __slots__ = ()

    @abstractmethod
    def send(self, value):
        ...

    @abstractmethod
    def throw(self, typ, val=None, tb=None):
        ...

    def close(self):
        ...
        
    @classmethod
    def __subclasshook__(cls, C):
        if cls is Coroutine:
            return _check_methods(C, '__await__', 'send', 'throw', 'close')
        return NotImplemented
```

接下来是异步生成器，来看一个例子：

假如我要到一家超市去购买土豆，而超市货架上的土豆数量是有限的：

```python
class Potato:
    @classmethod
    def make(cls, num, *args, **kws):
        potatos = []
        for i in range(num):
            potatos.append(cls.__new__(cls, *args, **kws))
        return potatos

all_potatos = Potato.make(5)
```

现在我想要买50个土豆，每次从货架上拿走一个土豆放到篮子：

```python
def take_potatos(num):
    count = 0
    while True:
        if len(all_potatos) == 0:
            sleep(.1)
        else:
            potato = all_potatos.pop()
            yield potato
            count += 1
            if count == num:
                break

def buy_potatos():
    bucket = []
    for p in take_potatos(50):
        bucket.append(p)
```

对应到代码中，就是迭代一个生成器的模型，显然，当货架上的土豆不够的时候，这时只能够死等，而且在上面例子中等多长时间都不会有结果（因为一切都是同步的），也许可以用多进程和多线程解决，而在现实生活中，更应该像是这样的：

```python
async def take_potatos(num):
    count = 0
    while True:
        if len(all_potatos) == 0:
            await ask_for_potato()
        potato = all_potatos.pop()
        yield potato
        count += 1
        if count == num:
            break
```

当货架上的土豆没有了之后，我可以询问超市请求需要更多的土豆，这时候需要等待一段时间直到生产者完成生产的过程：

```python
async def ask_for_potato():
    await asyncio.sleep(random.random())
    all_potatos.extend(Potato.make(random.randint(1, 10)))
```

当生产者完成和返回之后，这是便能从await挂起的地方继续往下跑，完成消费的过程。而这整一个过程，就是一个异步生成器迭代的流程：

```python
async def buy_potatos():
    bucket = []
    async for p in take_potatos(50):
        bucket.append(p)
        print(f'Got potato {id(p)}...')
```

async for语法表示我们要后面迭代的是一个异步生成器。

```python
def main():
    import asyncio
    loop = asyncio.get_event_loop()
    res = loop.run_until_complete(buy_potatos())
    loop.close()
```

用asyncio运行这段代码，结果是这样的：

```
Got potato 4338641384...
Got potato 4338641160...
Got potato 4338614736...
Got potato 4338614680...
Got potato 4338614568...
Got potato 4344861864...
Got potato 4344843456...
Got potato 4344843400...
Got potato 4338641384...
Got potato 4338641160...
...
```

既然是异步的，在请求之后不一定要死等，而是可以做其他事情。比如除了土豆，我还想买番茄，这时只需要在事件循环中再添加一个过程：

```python
def main():
    import asyncio
    loop = asyncio.get_event_loop()
    res = loop.run_until_complete(asyncio.wait([buy_potatos(), buy_tomatos()]))
    loop.close()
```

再来运行这段代码：

```
Got potato 4423119312...
Got tomato 4423119368...
Got potato 4429291024...
Got potato 4421640768...
Got tomato 4429331704...
Got tomato 4429331760...
Got tomato 4423119368...
Got potato 4429331760...
Got potato 4429331704...
Got potato 4429346688...
Got potato 4429346072...
Got tomato 4429347360...
...
```

看下AsyncGenerator的定义，它需要实现__aiter__和__anext__两个核心方法，以及asend，athrow，aclose方法。

```python
class AsyncGenerator(AsyncIterator):
    __slots__ = ()

    async def __anext__(self):
        ...

    @abstractmethod
    async def asend(self, value):
        ...

    @abstractmethod
    async def athrow(self, typ, val=None, tb=None):
        ...

    async def aclose(self):
        ...

    @classmethod
    def __subclasshook__(cls, C):
        if cls is AsyncGenerator:
            return _check_methods(C, '__aiter__', '__anext__',
                                  'asend', 'athrow', 'aclose')
        return NotImplemented
```

异步生成器是在3.6之后才有的特性，同样的还有异步推导表达式，因此在上面的例子中，也可以写成这样：

```python
bucket = [p async for p in take_potatos(50)]
```

类似的，还有await表达式：

```python
result = [await fun() for fun in funcs if await condition()]
```

除了函数之外，类实例的普通方法也能用async语法修饰：

```python
class ThreeTwoOne:
    async def begin(self):
        print(3)
        await asyncio.sleep(1)
        print(2)
        await asyncio.sleep(1)
        print(1)        
        await asyncio.sleep(1)
        return

async def game():
    t = ThreeTwoOne()
    await t.begin()
    print('start')
```

实例方法的调用同样是返回一个coroutine：

```python
function = ThreeTwoOne.begin
method = function.__get__(ThreeTwoOne, ThreeTwoOne())
import inspect
assert inspect.isfunction(function)
assert inspect.ismethod(method)
assert inspect.iscoroutine(method())
```

同理还有类方法：

```python
class ThreeTwoOne:
    @classmethod
    async def begin(cls):
        print(3)
        await asyncio.sleep(1)
        print(2)
        await asyncio.sleep(1)
        print(1)        
        await asyncio.sleep(1)
        return

async def game():
    await ThreeTwoOne.begin()
    print('start')
```

根据PEP 492中，async也可以应用到上下文管理器中，__aenter__和__aexit__需要返回一个Awaitable：

```python
class GameContext:
    async def __aenter__(self):
        print('game loading...')
        await asyncio.sleep(1)

    async def __aexit__(self, exc_type, exc, tb):
        print('game exit...')
        await asyncio.sleep(1)

async def game():
    async with GameContext():
        print('game start...')
        await asyncio.sleep(2)
```

在3.7版本，contextlib中会新增一个asynccontextmanager装饰器来包装一个实现异步协议的上下文管理器：

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_connection():
    conn = await acquire_db_connection()
    try:
        yield
    finally:
        await release_db_connection(conn)
```

async修饰符也能用在`__call__`方法上：

```python
class GameContext:
    async def __aenter__(self):
        self._started = time()
        print('game loading...')
        await asyncio.sleep(1)
        return self

    async def __aexit__(self, exc_type, exc, tb):
        print('game exit...')
        await asyncio.sleep(1)

    async def __call__(self, *args, **kws):
        if args[0] == 'time':
            return time() - self._started

async def game():
    async with GameContext() as ctx:
        print('game start...')
        await asyncio.sleep(2)
        print('game time: ', await ctx('time'))
```

**await和yield from**

Python3.3的yield from语法可以把生成器的操作委托给另一个生成器，生成器的调用方可以直接与子生成器进行通信：

```python
def sub_gen():
    yield 1
    yield 2
    yield 3

def gen():
    return (yield from sub_gen())

def main():
    for val in gen():
        print(val)
# 1
# 2
# 3
```

利用这一特性，使用yield from能够编写出类似协程效果的函数调用，在3.5之前，asyncio正是使用@asyncio.coroutine和yield from语法来创建协程：

```python
# https://docs.python.org/3.4/library/asyncio-task.html
import asyncio

@asyncio.coroutine
def compute(x, y):
    print("Compute %s + %s ..." % (x, y))
    yield from asyncio.sleep(1.0)
    return x + y

@asyncio.coroutine
def print_sum(x, y):
    result = yield from compute(x, y)
    print("%s + %s = %s" % (x, y, result))

loop = asyncio.get_event_loop()
loop.run_until_complete(print_sum(1, 2))
loop.close()
```

然而，用yield from容易在表示协程和生成器中混淆，没有良好的语义性，所以在Python 3.5推出了更新的async/await表达式来作为协程的语法。

因此类似以下的调用是等价的：

```python
async with lock:
    ...
    
with (yield from lock):
    ...
######################
def main():
    return (yield from coro())

def main():
    return (await coro())
```

那么，怎么把生成器包装为一个协程对象呢？这时候可以用到types包中的coroutine装饰器（如果使用asyncio做驱动的话，那么也可以使用asyncio的coroutine装饰器），`@types.coroutine`装饰器会将一个生成器函数包装为协程对象：

```python
import asyncio
import types

@types.coroutine
def compute(x, y):
    print("Compute %s + %s ..." % (x, y))
    yield from asyncio.sleep(1.0)
    return x + y

async def print_sum(x, y):
    result = await compute(x, y)
    print("%s + %s = %s" % (x, y, result))

loop = asyncio.get_event_loop()
loop.run_until_complete(print_sum(1, 2))
loop.close()
```

尽管两个函数分别使用了新旧语法，但他们都是协程对象，也分别称作*native coroutine*以及*generator-based coroutine*，因此不用担心语法问题。

下面观察一个asyncio中Future的例子：

```python
import asyncio

future = asyncio.Future()

async def coro1():
    await asyncio.sleep(1)
    future.set_result('data')

async def coro2():
    print(await future)

loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait([
    coro1(), 
    coro2()
]))
loop.close()
```

两个协程在在事件循环中，协程coro1在执行第一句后挂起自身切到asyncio.sleep，而协程coro2一直等待future的结果，让出事件循环，计时器结束后coro1执行了第二句设置了future的值，被挂起的coro2恢复执行，打印出future的结果'data'。

future可以被await证明了future对象是一个Awaitable，进入Future类的源码可以看到有一段代码显示了future实现了`__await__`协议：

```python
class Future:
    ...
    def __iter__(self):
        if not self.done():
            self._asyncio_future_blocking = True
            yield self  # This tells Task to wait for completion.
        assert self.done(), "yield from wasn't used with future"
        return self.result()  # May raise too.

    if compat.PY35:
        __await__ = __iter__ # make compatible with 'await' expression
```

当执行**await future**这行代码时，future中的这段代码就会被执行，首先future检查它自身是否已经完成，如果没有完成，挂起自身，告知当前的Task（任务）等待future完成。

当future执行set_result方法时，会触发以下的代码，设置结果，标记future已经完成：

```python
def set_result(self, result):
    ...
    if self._state != _PENDING:
        raise InvalidStateError('{}: {!r}'.format(self._state, self))
    self._result = result
    self._state = _FINISHED
    self._schedule_callbacks()
```

最后future会调度自身的回调函数，触发Task._step()告知Task驱动future从之前挂起的点恢复执行，不难看出，future会执行下面的代码：

```python
class Future:
    ...
    def __iter__(self):
        ...
        assert self.done(), "yield from wasn't used with future"
        return self.result()  # May raise too.
```

最终返回结果给调用方。

前面讲了那么多关于asyncio的例子，那么除了asyncio，就没有其他协程库了吗？asyncio作为python的标准库，自然受到很多青睐，但它有时候还是显得太重量了，尤其是提供了许多复杂的轮子和协议，不便于使用。

你可以理解为，asyncio是使用async/await语法开发的协程库，而不是有asyncio才能用async/await，除了asyncio之外，curio和trio是更加轻量级的替代物，而且也更容易使用。

curio的作者是David Beazley，下面是使用curio创建tcp server的例子，据说这是dabeaz理想中的一个异步服务器的样子：

```python
from curio import run, spawn
from curio.socket import *

async def echo_server(address):
    sock = socket(AF_INET, SOCK_STREAM)
    sock.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
    sock.bind(address)
    sock.listen(5)
    print('Server listening at', address)
    async with sock:
        while True:
            client, addr = await sock.accept()
            await spawn(echo_client, client, addr)

async def echo_client(client, addr):
    print('Connection from', addr)
    async with client:
         while True:
             data = await client.recv(100000)
             if not data:
                 break
             await client.sendall(data)
    print('Connection closed')

if __name__ == '__main__':
    run(echo_server, ('',25000))
```

无论是asyncio还是curio，或者是其他异步协程库，在背后往往都会借助于IO的事件循环来实现异步，下面用几十行代码来展示一个简陋的基于事件驱动的echo服务器：

```python
from socket import socket, AF_INET, SOCK_STREAM, SOL_SOCKET, SO_REUSEADDR
from selectors import DefaultSelector, EVENT_READ

selector = DefaultSelector()
pool = {}

def request(client_socket, addr):
    client_socket, addr = client_socket, addr
    def handle_request(key, mask):
        data = client_socket.recv(100000)
        if not data:
            client_socket.close()
            selector.unregister(client_socket)
            del pool[addr]
        else:
            client_socket.sendall(data)
    return handle_request

def recv_client(key, mask):
    sock = key.fileobj
    client_socket, addr = sock.accept()
    req = request(client_socket, addr)
    pool[addr] = req
    selector.register(client_socket, EVENT_READ, req)

def echo_server(address):
    sock = socket(AF_INET, SOCK_STREAM)
    sock.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)
    sock.bind(address)
    sock.listen(5)
    selector.register(sock, EVENT_READ, recv_client)
    try:
        while True:
            events = selector.select()
            for key, mask in events:
                callback = key.data
                callback(key, mask)
    except KeyboardInterrupt:
        sock.close()

if __name__ == '__main__':
    echo_server(('',25000))
```

验证一下：

```
# terminal 1
$ nc localhost 25000
hello world
hello world

# terminal 2
$ nc localhost 25000
hello world
hello world
```

现在知道，完成异步的代码不一定要用async/await，使用了async/await的代码也不一定能做到异步，async/await是协程的语法糖，使协程之间的调用变得更加清晰，使用async修饰的函数调用时会返回一个协程对象，await只能放在async修饰的函数里面使用，await后面必须要跟着一个协程对象或Awaitable，await的目的是等待协程控制流的返回，而实现暂停并挂起函数的操作是yield。

个人认为，async/await以及协程是Python未来实现异步编程的趋势，我们将会在更多的地方看到他们的身影，例如协程库的curio和trio，web框架的sanic，数据库驱动的asyncpg等等...在Python 3主导的今天，作为开发者，应该及时拥抱和适应新的变化，而基于async/await的协程凭借良好的可读性和易用性日渐登上舞台，看到这里，你还不赶紧上车？

参考：

[PEP 492](https://www.python.org/dev/peps/pep-0492/%23new-standard-library-functions)

[PEP 525](https://www.python.org/dev/peps/pep-0525/%23asynchronous-generators)

