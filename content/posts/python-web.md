+++
title = "从0到1，Python Web开发的进击之路"
date = 2017-02-28T00:00:00+08:00
tags = ["python", "web"]
categories = [""]
draft = false
+++

本文将以个人（开发）的角度，讲述如何从零开始，编写、搭建和部署一个基于Python的Web应用程序。

从最简单的出发点来剖析，一个web应用后端要完成的工作抽象出来无非就是3点：

1. 接收和解析请求。
2. 处理业务逻辑。
3. 生产和返回响应。

对于初学者来说，我们关心的只需这些步骤就够了。要检验这三个步骤，最简单的方法是先写出一个hello world。

```
request->"hello world"->response
```

python有许多流行的web框架，我们该如何选择呢？试着考虑三个因素：

- 易用：该框架是面对初学者友好的，而且具有健全的文档，灵活开发部署。例如flask，bottle。
- 效率：该框架适合快速开发，拥有丰富的轮子，注重开发效率。例如django。
- 性能：该框架能承载更大的请求压力，提高吞吐量。例如falcon，tornado，aiohttp，sanic。

根据场景使用合适的框架能少走许多弯路，当然，你还能自己写一个框架，这个下面再说。

对于缺乏经验的人来说，易用性无疑是排在第一位的，推荐用flask作为python web入门的第一个框架，另外也推荐django。

首先用virtualenv创建python的应用环境，为什么用virtualenv呢，virtualenv能创建一个纯净独立的python环境，避免污染全局环境。（顺便安利kennethreitz大神的[pipenv](https://github.com/kennethreitz/pipenv)）

```shell
mkdir todo
cd todo
virtualenv venv
source venv/bin/activate
pip install flask
touch server.py
```

代码未写，规范先行。在写代码之前要定义好一套良好代码规范，例如PEP8。这样才能使得你的代码变的更加可控。

心中默念The Zen of Python：

```
Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Although that way may not be obvious at first unless you're Dutch.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
```

下面用flask来编写第一个程序：

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/index')
def index():
    return jsonify(msg='hello world')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

在命令行输入python server.py

```shell
python server.py
* Running on http://0.0.0.0:8000/ (Press CTRL+C to quit)
```

打开浏览器，访问[http://127.0.0.1:8000/index](http://127.0.0.1%3A8000/index%25EF%25BC%258C%25E5%25A6%2582%25E6%2597%25A0%25E6%2584%258F%25E5%25A4%2596%25EF%25BC%258C%25E4%25BC%259A%25E7%259C%258B%25E5%2588%25B0%25E4%25B8%258B%25E9%259D%25A2%25E7%259A%2584%25E5%2593%258D%25E5%25BA%2594%25E3%2580%2582)，如无意外，会看到下面的响应。

```json
{
  "msg": "hello world"
}
```

一个最简单的web程序就完成了！让我们看下过程中都发生了什么：

1. 客户端（浏览器）根据输入的地址[http://127.0.0.1:8000/index](http://127.0.0.1%3A8000/index%25E6%2589%25BE%25E5%2588%25B0%25E5%258D%258F%25E8%25AE%25AE%25EF%25BC%2588http%29%25EF%25BC%258C%25E4%25B8%25BB%25E6%259C%25BA%25EF%25BC%2588127.0.0.1%25EF%25BC%2589%25EF%25BC%258C%25E7%25AB%25AF%25E5%258F%25A3%25EF%25BC%25888000%25EF%25BC%2589%25E5%2592%258C%25E8%25B7%25AF%25E5%25BE%2584%25EF%25BC%2588/index%25EF%25BC%2589%25EF%25BC%258C%25E4%25B8%258E%25E6%259C%258D%25E5%258A%25A1%25E5%2599%25A8%25EF%25BC%2588application)找到协议（http)，主机（127.0.0.1），端口（8000）和路径（/index），与服务器（application server）建立三次握手，并发送一个http请求。

2. 服务器（application server）把请求报文封装成请求对象，根据路由（router）找到/index这个路径所对应的视图函数，调用这个视图函数。

3. 视图函数生成一个http响应，返回一个json数据给客户端。

   ```
   HTTP/1.0 200 OK
   Content-Type: application/json
   Content-Length: 27
   Server: Werkzeug/0.11.15 Python/3.5.2
   Date: Thu, 26 Jan 2017 05:14:36 GMT
   ```

当我们输入python server.py时，会建立一个服务器（也叫应用程序服务器，即application server）来监听请求，并把请求转给flask来处理。那么这个服务器是如何跟python程序打交道的呢？答案就是[WSGI](http://archimedeanco.com/wsgi-tutorial/%23)(Web Server Gateway Interface)接口，它是server端（服务器）与application端（应用程序）之间的一套约定俗成的规范，使我们只要编写一个统一的接口，就能应用到不同的wsgi server上。用图表示它们的关系，就是下面这样的：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-4f9e190627003e1b56fab93afba5f3ca_1440w.png)

只要application端（flask）和server端（flask内建的server）都遵循wsgi这个规范，那么他们就能够协同工作了，关于WSGI规范，可参阅Python官方的[PEP 333](https://www.python.org/dev/peps/pep-0333/)里的说明。

目前为止，应用是下面这个样子的：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-73e7f74fc237221f65dc3fe0cb30a911_1440w.png)

一切都很简单，现在我们要做一个Todo应用，提供添加todo，修改todo状态和删除todo的接口。

先不考虑数据库，可以迅速地写出下面的代码：

```python
from flask import Flask, jsonify, request, abort, Response
from time import time
from uuid import uuid4
import json

app = Flask(__name__)

class Todo(object):
    def __init__(self, content):
        self.id = str(uuid4())
        self.content = content #todo内容
        self.created_at = time() #创建时间
        self.is_finished = False #是否完成
        self.finished_at = None #完成时间

    def finish(self):
        self.is_finished = True
        self.finished_at = time()

    def json(self):
        return {
            'id': self.id,
            'content': self.content,
            'created_at': self.created_at,
            'is_finished': self.is_finished,
            'finished_at': self.finished_at
        }

todos = {}
get_todo = lambda tid: todos.get(tid, False)

@app.route('/todo')
def index():
    return jsonify(data=[todo.json() for todo in todos.values()])

@app.route('/todo', methods=['POST'])
def add():
    content = request.form.get('content', None)
    if not content:
        abort(400)
    todo = Todo(content)
    todos[todo.id] = todo
    return Response() #200

@app.route('/todo/<tid>/finish', methods=['PUT'])
def finish(tid):
    todo = get_todo(tid)
    if todo:
        todo.finish()
        todos[todo.id] = todo
        return Response()
    abort(404)

@app.route('/todo/<tid>', methods=['DELETE'])
def delete(tid):
    todo = get_todo(tid)
    if todo:
        todos.pop(tid)
        return Response()
    abort(404)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

这个程序基本实现了需要的接口，现在测试一下功能。

- 添加一个todo

```
http -f POST http://127.0.0.1:8000/todo content=好好学习
HTTP/1.0 200 OK
Content-Length: 0
Content-Type: text/html; charset=utf-8
Date: Thu, 26 Jan 2017 06:45:37 GMT
Server: Werkzeug/0.11.15 Python/3.5.2
```

- 查看todo列表

```
http http://127.0.0.1:8000/todo
HTTP/1.0 200 OK
Content-Length: 203
Content-Type: application/json
Date: Thu, 26 Jan 2017 06:46:16 GMT
Server: Werkzeug/0.11.15 Python/3.5.2

{
    "data": [
        "{\"created_at\": 1485413137.305699, \"id\": \"6f2b28c4-1e83-45b2-8b86-20e28e21cd40\", \"is_finished\": false, \"finished_at\": null, \"content\": \"\\u597d\\u597d\\u5b66\\u4e60\"}"
    ]
}
```

- 修改todo状态

```
http -f PUT http://127.0.0.1:8000/todo/6f2b28c4-1e83-45b2-8b86-20e28e21cd40/finish
HTTP/1.0 200 OK
Content-Length: 0
Content-Type: text/html; charset=utf-8
Date: Thu, 26 Jan 2017 06:47:18 GMT
Server: Werkzeug/0.11.15 Python/3.5.2

http http://127.0.0.1:8000/todo
HTTP/1.0 200 OK
Content-Length: 215
Content-Type: application/json
Date: Thu, 26 Jan 2017 06:47:22 GMT
Server: Werkzeug/0.11.15 Python/3.5.2

{
    "data": [
        "{\"created_at\": 1485413137.305699, \"id\": \"6f2b28c4-1e83-45b2-8b86-20e28e21cd40\", \"is_finished\": true, \"finished_at\": 1485413238.650981, \"content\": \"\\u597d\\u597d\\u5b66\\u4e60\"}"
    ]
}
```

- 删除todo

```
http -f DELETE http://127.0.0.1:8000/todo/6f2b28c4-1e83-45b2-8b86-20e28e21cd40
HTTP/1.0 200 OK
Content-Length: 0
Content-Type: text/html; charset=utf-8
Date: Thu, 26 Jan 2017 06:48:20 GMT
Server: Werkzeug/0.11.15 Python/3.5.2

http http://127.0.0.1:8000/todo
HTTP/1.0 200 OK
Content-Length: 17
Content-Type: application/json
Date: Thu, 26 Jan 2017 06:48:22 GMT
Server: Werkzeug/0.11.15 Python/3.5.2

{
    "data": []
}
```

但是这个的程序的数据都保存在内存里，只要服务一停止所有的数据就没办法保存下来了，因此，我们还需要一个数据库用于持久化数据。

那么，应该选择什么数据库呢？

- 传统的rdbms，例如mysql，postgresql等，他们具有很高的稳定性和不俗的性能，结构化查询，支持事务，由ACID来保持数据的完整性。
- nosql，例如mongodb，cassandra等，他们具有非结构化特性，易于横向扩展，实现数据的自动分片，拥有灵活的存储结构和强悍的读写性能。

这里使用mongodb作例子，使用mongodb改造后的代码是这样的：

```python
from flask import Flask, jsonify, request, abort, Response
from time import time
from bson.objectid import ObjectId
from bson.json_util import dumps
import pymongo

app = Flask(__name__)

mongo = pymongo.MongoClient('127.0.0.1', 27017)
db = mongo.todo

class Todo(object):
    @classmethod
    def create_doc(cls, content):
        return {
            'content': content,
            'created_at': time(),
            'is_finished': False,
            'finished_at': None
        }

@app.route('/todo')
def index():
    todos = db.todos.find({})
    return dumps(todos)

@app.route('/todo', methods=['POST'])
def add():
    content = request.form.get('content', None)
    if not content:
        abort(400)
    db.todos.insert(Todo.create_doc(content))
    return Response() #200

@app.route('/todo/<tid>/finish', methods=['PUT'])
def finish(tid):
    result = db.todos.update_one(
        {'_id': ObjectId(tid)},
        {
            '$set': {
                'is_finished': True,
                'finished_at': time()
            }
        }    
    )
    if result.matched_count == 0:
        abort(404)
    return Response()

@app.route('/todo/<tid>', methods=['DELETE'])
def delete(tid):
    result = db.todos.delete_one(
        {'_id': ObjectId(tid)}  
    )
    if result.matched_count == 0:
        abort(404)
    return Response()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

这样一来，应用的数据便能持久化到本地了。现在，整个应用看起来是下面这样的：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-0aa027a5a04db7ac2245f42f39b94f78_1440w.png)

现在往mongodb插入1万条数据。

```python
import requests

for i in range(10000):
    requests.post('http://127.0.0.1:8000/todo', {'content': str(i)})
```

获取todo的接口目前是有问题的，因为它一次性把数据库的所有记录都返回了，当数据记录增长到一万条的时候，这个接口的请求就会变的非常慢，需要500ms后才能发出响应。现在对它进行如下的改造：

```python
@app.route('/todo')
def index():
    start = request.args.get('start', '')
    start = int(start) if start.isdigit() else 0
    todos = db.todos.find().sort([('created_at', -1)]).limit(10).skip(start)
    return dumps(todos)
```

每次只取十条记录，按创建日期排序，先取最新的，用分页的方式获取以往记录。改造后的接口现在只需50ms便能返回响应。

现在对这个接口进行性能测试：

```
wrk -c 100 -t 12 -d 5s http://127.0.0.1:8000/todo
Running 5s test @ http://127.0.0.1:8000/todo
  12 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.22s   618.29ms   1.90s    48.12%
    Req/Sec    14.64     10.68    40.00     57.94%
  220 requests in 5.09s, 338.38KB read
  Socket errors: connect 0, read 0, write 0, timeout 87
Requests/sec:     43.20
Transfer/sec:     66.45KB
```

rps只有43。我们继续进行改进，通过观察我们发现我们查询todo时需要通过created_at这个字段进行排序再过滤，这样以来每次查询都要先对10000条记录进行排序，效率自然变的很低，对于这个场景，可以对created_at这个字段做索引：

```
db.todos.ensureIndex({'created_at': -1})
```

通过explain我们轻易地看出mongo使用了索引做扫描

```javascript
> db.todos.find().sort({'created_at': -1}).limit(10).explain()
/* 1 */
{
    "queryPlanner" : {
        "plannerVersion" : 1,
        "namespace" : "todo.todos",
        "indexFilterSet" : false,
        "parsedQuery" : {},
        "winningPlan" : {
            "stage" : "LIMIT",
            "limitAmount" : 10,
            "inputStage" : {
                "stage" : "FETCH",
                "inputStage" : {
                    "stage" : "IXSCAN",
                    "keyPattern" : {
                        "created_at" : -1.0
                    },
                    "indexName" : "created_at_-1",
                    "isMultiKey" : false,
                    "multiKeyPaths" : {
                        "created_at" : []
                    },
                    "isUnique" : false,
                    "isSparse" : false,
                    "isPartial" : false,
                    "indexVersion" : 2,
                    "direction" : "forward",
                    "indexBounds" : {
                        "created_at" : [ 
                            "[MaxKey, MinKey]"
                        ]
                    }
                }
            }
        },
        "rejectedPlans" : []
    },
    "serverInfo" : {
        "host" : "841bf506b6ec",
        "port" : 27017,
        "version" : "3.4.1",
        "gitVersion" : "5e103c4f5583e2566a45d740225dc250baacfbd7"
    },
    "ok" : 1.0
}
```

现在再做一轮性能测试，有了索引之后就大大降低了排序的成本，rps提高到了298。

```
wrk -c 100 -t 12 -d 5s http://127.0.0.1:8000/todo
Running 5s test @ http://127.0.0.1:8000/todo
  12 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   310.32ms   47.51ms 357.47ms   94.57%
    Req/Sec    26.88     14.11    80.00     76.64%
  1511 requests in 5.06s, 2.27MB read
Requests/sec:    298.34
Transfer/sec:    458.87KB
```

再把重心放到app server上，目前我们使用flask内建的wsgi server，这个server由于是单进程单线程模型的，所以性能很差，一个请求不处理完的话服务器就会阻塞住其他请求，我们需要对这个server做替换。关于python web的app server选择，目前主流采用的有：

- gunicorn
- uWSGI

我们看[gunicorn](http://www.jianshu.com/p/docs.gunicorn.org/en/latest/design.html)文档可以得知，gunicorn是一个python编写的高效的WSGI HTTP服务器，gunicorn使用pre-fork模型（一个master进程管理多个child子进程），使用gunicorn的方法十分简单：

```
gunicorn --workers=9 server:app --bind 127.0.0.1:8000
```

根据文档说明使用（2 * cpu核心数量）+1个worker，还要传入一个兼容wsgi app的start up方法，通过Flask的源码可以看到，Flask这个类实现了下面这个接口：

```python
    def __call__(self, environ, start_response):
        """Shortcut for :attr:`wsgi_app`."""
        return self.wsgi_app(environ, start_response)
```

也就是说我们只需把flask实例的名字传给gunicorn就ok了：

```
gunicorn --workers=9 server:app --bind 127.0.0.1:8000
[2017-01-27 11:20:01 +0800] [5855] [INFO] Starting gunicorn 19.6.0
[2017-01-27 11:20:01 +0800] [5855] [INFO] Listening at: http://127.0.0.1:8000 (5855)
[2017-01-27 11:20:01 +0800] [5855] [INFO] Using worker: sync
[2017-01-27 11:20:01 +0800] [5889] [INFO] Booting worker with pid: 5889
[2017-01-27 11:20:01 +0800] [5890] [INFO] Booting worker with pid: 5890
[2017-01-27 11:20:01 +0800] [5891] [INFO] Booting worker with pid: 5891
[2017-01-27 11:20:01 +0800] [5892] [INFO] Booting worker with pid: 5892
[2017-01-27 11:20:02 +0800] [5893] [INFO] Booting worker with pid: 5893
[2017-01-27 11:20:02 +0800] [5894] [INFO] Booting worker with pid: 5894
[2017-01-27 11:20:02 +0800] [5895] [INFO] Booting worker with pid: 5895
[2017-01-27 11:20:02 +0800] [5896] [INFO] Booting worker with pid: 5896
[2017-01-27 11:20:02 +0800] [5897] [INFO] Booting worker with pid: 5897
```

可以看到gunicorn启动了9个进程（其中1个父进程）监听请求。使用了多进程的模型看起来是下面这样的：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-7ecc8340792bd7009cd02f41e092605d_1440w.png)

继续进行性能测试，可以看到吞吐量又有了很大的提升：

```
wrk -c 100 -t 12 -d 5s http://127.0.0.1:8000/todo
Running 5s test @ http://127.0.0.1:8000/todo
  12 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   109.30ms   16.10ms 251.01ms   90.31%
    Req/Sec    72.47     10.48   100.00     78.89%
  4373 requests in 5.07s, 6.59MB read
Requests/sec:    863.35
Transfer/sec:      1.30MB
```

那么gunicorn还能再优化吗，答案是肯定的。回到之前我们发现了这一行：

```
[2017-01-27 11:20:01 +0800] [5855] [INFO] Using worker: sync
```

也就是说，gunicorn worker使用的是sync（同步）模式来处理请求，那么它支持async（异步）模式吗，再看gunicorn的文档有下面一段说明：

```
Async Workers
The asynchronous workers available are based on Greenlets (via Eventlet and Gevent). Greenlets are an implementation of cooperative multi-threading for Python. In general, an application should be able to make use of these worker classes with no changes.
```

gunicorn支持基于greenlet的异步的worker，它使得worker能够协作式地工作。当worker阻塞在外部调用的IO操作时，gunicorn会聪明地把执行调度给其他worker，挂起当前的worker，直至IO操作完成后，被挂起的worker又会重新加入到调度队列中，这样gunicorn便有能力处理大量的并发请求了。

gunicorn有两个不错的async worker：

- meinheld
- gevent

[meinheld](https://github.com/mopemope/meinheld)是一个基于picoev的异步WSGI Web服务器，它可以很轻松地集成到gunicorn中，处理wsgi请求。

```
gunicorn --workers=9 --worker-class="meinheld.gmeinheld.MeinheldWorker" server:app --bind 127.0.0.1:8000
[2017-01-27 11:47:01 +0800] [7497] [INFO] Starting gunicorn 19.6.0
[2017-01-27 11:47:01 +0800] [7497] [INFO] Listening at: http://127.0.0.1:8000 (7497)
[2017-01-27 11:47:01 +0800] [7497] [INFO] Using worker: meinheld.gmeinheld.MeinheldWorker
[2017-01-27 11:47:01 +0800] [7531] [INFO] Booting worker with pid: 7531
[2017-01-27 11:47:01 +0800] [7532] [INFO] Booting worker with pid: 7532
[2017-01-27 11:47:01 +0800] [7533] [INFO] Booting worker with pid: 7533
[2017-01-27 11:47:01 +0800] [7534] [INFO] Booting worker with pid: 7534
[2017-01-27 11:47:01 +0800] [7535] [INFO] Booting worker with pid: 7535
[2017-01-27 11:47:01 +0800] [7536] [INFO] Booting worker with pid: 7536
[2017-01-27 11:47:01 +0800] [7537] [INFO] Booting worker with pid: 7537
[2017-01-27 11:47:01 +0800] [7538] [INFO] Booting worker with pid: 7538
[2017-01-27 11:47:01 +0800] [7539] [INFO] Booting worker with pid: 7539
```

可以看到现在使用的是meinheld.gmeinheld.MeinheldWorker这个worker。再进行性能测试看看：

```
wrk -c 100 -t 12 -d 5s http://127.0.0.1:8000/todo
Running 5s test @ http://127.0.0.1:8000/todo
  12 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    84.53ms   39.90ms 354.42ms   72.11%
    Req/Sec    94.52     20.84   150.00     70.28%
  5684 requests in 5.04s, 8.59MB read
Requests/sec:   1128.72
Transfer/sec:      1.71MB
```

果然提升了不少。

现在有了app server，那需要nginx之类的web server吗？看看[nginx](http://www.aosabook.org/en/nginx.html)反向代理能带给我们什么好处：

- 负载均衡，把请求平均地分到上游的app server进程。
- 静态文件处理，静态文件的访问交给nginx来处理，降低了app server的压力。
- 接收完客户端所有的TCP包，再一次交给上游的应用来处理，防止app server被慢请求干扰。
- 访问控制和路由重写。
- 强大的ngx_lua模块。
- Proxy cache。
- Gzip，SSL...

为了以后的扩展性，带上一个nginx是有必要的，但如果你的应用没大的需求，那么可加可不加。

想让nginx反向代理gunicorn，只需对nginx的配置文件加入几行配置，让nginx通过proxy_pass打到gunicorn监听的端口上就可以了：

```nginx
      server {
          listen 8888;

          location / {
               proxy_pass http://127.0.0.1:8000;
               proxy_redirect     off;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          }
      }
```

现在应用的结构是这样的：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-86eb1b5263ae630d097e1173e3848032_1440w.png)

但仅仅是这样还是不足以应对高并发下的请求的，洪水般的请求势必是对数据库的一个重大考验，把请求数提升到1000，出现了大量了timeout：

```
wrk -c 1000 -t 12 -d 5s http://127.0.0.1:8888/todo
Running 5s test @ http://127.0.0.1:8888/todo
  12 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   239.50ms  235.76ms   1.93s    91.13%
    Req/Sec    83.07     76.77   434.00     76.78%
  4548 requests in 5.10s, 6.52MB read
  Socket errors: connect 0, read 297, write 0, timeout 36
  Non-2xx or 3xx responses: 289
Requests/sec:    892.04
Transfer/sec:      1.28MB
```

阻止洪峰的方法有：

- 限流（水桶算法）
- 分流（负载均衡）
- 缓存
- 访问控制

等等..这里重点说缓存，缓存系统是每个web应用程序重要的一个模块，缓存的作用是把热点数据放入内存中，降低对数据库的压力。

下面用redis来对第一页的数据进行缓存：

```python
rds = redis.StrictRedis('127.0.0.1', 6379)

@app.route('/todo')
def index():
    start = request.args.get('start', '')
    start = int(start) if start.isdigit() else 0
    data = rds.get('todos')
    if data and start == 0:
        return data
    todos = db.todos.find().sort([('created_at', -1)]).limit(10).skip(start)
    data = dumps(todos)
    rds.set('todos', data, 3600)
    return data
```

只有在第一次请求时接触到数据库，其余请求都会从缓存中读取，瞬间就提高了应用的rps。

```
wrk -c 1000 -t 12 -d 5s http://127.0.0.1:8888/todo
Running 5s test @ http://127.0.0.1:8888/todo
  12 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    68.33ms   95.27ms   1.34s    93.69%
    Req/Sec   277.32    258.20     1.60k    77.33%
  15255 requests in 5.10s, 22.77MB read
  Socket errors: connect 0, read 382, write 0, timeout 0
  Non-2xx or 3xx responses: 207
Requests/sec:   2992.79
Transfer/sec:      4.47MB
```

上面的这个示例只展示了基础的缓存方式，并没有针对多用户的情况处理，在涉及到状态条件的影响下，应该使用更加复杂的缓存策略。

现在再来考虑使用缓存不当会造成几个问题，设置缓存的时间是3600秒，当3600秒过后缓存失效，而新缓存又没完成的中间时间内，如果有大量请求到来，就会蜂拥去查询数据库，这种现象称为**缓存雪崩**，针对这个情况，可以对数据库请求这个动作进行加锁，只允许第一个请求访问数据库，更新缓存后其他的请求都会访问缓存，第二种方法是做二级缓存，拷贝缓存比一级缓存设置更长的过期时间。还有**缓存穿透**和**缓存一致性**等问题虽然这里没有体现，但也是缓存设计中值得思考的几个点。

下面是加入缓存后的系统结构：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-207c6dfdd3777b49c0a5b507ec4886fb_1440w.png)

目前为止还不能说完善，如果中间某个进程挂掉了，那么整个系统的稳定性就会土崩瓦解。为此，要在中间加入一个进程管理工具：supervisor来监控和重启应用进程。

首先要建立supervisor的配置文件：supervisord.conf

```
[program:gunicorn]
command=gunicorn --workers=9 --worker-class="meinheld.gmeinheld.MeinheldWorker" server:app --bind 127.0.0.1:8000
autostart=true
autorestart=true
stdout_logfile=access.log
stderr_logfile=error.log
```

然后启动supervisord作为后台进程。

```
supervisord -c supervisord.conf
```

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-a165341ad880a5bcb1e2bd8d6623d800_1440w.png)

虽然缓存可以有效地帮我们减轻数据库的压力，但如果系统遇到大量并发的耗时任务时，进程也会阻塞在任务的处理上，影响了其他普通请求的正常响应，严重时，系统很可能会出现假死现象，为了针对对耗时任务的处理，我们的应用还需要引入一个外部作业的处理系统，当程序接收到耗时任务的请求时，交给任务的工作进程池来处理，然后再通过异步回调或消息通知等方式来获得处理结果。

应用程序与任务进程的通信通常借助消息队列的方式来进行通信，简单来说，应用程序会把任务信息序列化为一条消息（message）放入（push）与特定任务进程之间的信道里（channel），消息的中间件（broker）负责把消息持久化到存储系统，此时任务进程再通过轮询的方式获取消息，处理任务，再把结果存储和返回。

显然，此时我们需要一个负责分发消息和与队列打交道的调度器和一个存储消息的中间件。

Celery是基于Python的一个分布式的消息队列调度系统，我们把Celery作为消息调度器，Redis作为消息存储器，那么应用看起来应该是这样的。

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-38e3f9f636f1e4c80f6bdbe4093a4d0e_1440w.png)

一般来说，这个结构已经满足大多数的小规模应用了，剩下做的就是代码和组件配置的调优了。

然后还有一个很重要的点就是：测试

虽然很多人不喜欢写测试（我也不喜欢），但良好的测试对调试和排错是有很大帮助的。这里指的测试不仅仅是单元测试，关于测试可以从几个方面入手：

- 压力测试
  - wrk（请求）
  - htop（cpu和内存占用）
  - dstat（硬盘读写）
  - tcpdump（网络包）
  - iostat（io读写）
  - netstat（网络连接）
- 代码测试
  - unittest（单元测试）
  - selenium（浏览器测试）
  - mock/stub
- 黑盒测试
- 功能测试
- ...

还有另一个没提到的点就是：安全，主要注意几个点，其他奇形怪状的坑根据实际情况作相应的调整：

- SQL注入
- XSS攻击
- CSRF攻击
- 重要信息加密
- HTTPS
- 防火墙
- 访问控制

面对系统日益复杂和增长的依赖，有什么好的方法来保持系统的高可用和稳定一致性呢？docker是应用隔离和自动化构建的最佳选择。docker提供了一个抽象层，虚拟化了操作系统环境，用容器技术为应用的部署提供了沙盒环境，使应用之间能灵活地组合部署。

将每个组件独立为一个docker容器：

```
docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS                                                NAMES
cdca11112543        nginx               "nginx -g 'daemon off"   2 days ago          Exited (128) 2 days ago                                                         nginx
83119f92104a        cassandra           "/docker-entrypoint.s"   2 days ago          Exited (0) 2 days ago                                                           cassandra
841bf506b6ec        mongo               "/entrypoint.sh mongo"   2 days ago          Exited (1) 2 minutes ago   0.0.0.0:27017->27017/tcp, 0.0.0.0:28017->28017/tcp   mongo
b110a4530c4a        python:latest       "python3"                2 days ago          Exited (0) 46 hours ago                                                         python
b522b2a8313b        phpfpm              "docker-php-entrypoin"   4 days ago          Exited (0) 4 days ago                                                           php-fpm
f8630d4b48d7        spotify/kafka       "supervisord -n"         2 weeks ago         Exited (0) 6 days ago                                                           kafka
```

关于docker的用法，可以了解官方文档。

当业务逐渐快速增长时，原有的架构很可能已经不能支撑大流量带来的访问压力了。

这时候就可以使用进一步的方法来优化应用：

- **优化查询**，用数据库工具做explain，记录慢查询日志。
- **读写分离**，主节点负责接收写，复制的从节点负责读，并保持与主节点同步数据。
- **页面缓存**，如果你的应用是面向页面的，可以对页面和数据片进行缓存。
- **做冗余表**，把一些小表合并成大表以减少对数据库的查询次数。
- **编写C扩展**，把性能痛点交给C来处理。
- **提高机器配置**，这也许是最简单粗暴地提升性能的方式了...
- **PyPy**

不过即使再怎么优化，单机所能承载的压力毕竟是有限的，这时候就要引入更多的服务器，做LVS负载均衡，提供更大的负载能力。但多机器的优化带来了许多额外的问题，比如，机器之间怎么共享状态和数据，怎么通信，怎么保持一致性，所有的这些都迫使着原有的结构要更进一步，进化为一个分布式系统，使各个组件之间连接为一个互联网络。

这时候你需要解决的问题就更多了：

- 集群的部署搭建
- 单点问题
- 分布式锁
- 数据一致性
- 数据可用性
- 分区容忍性
- 数据备份
- 数据分片
- 数据热启动
- 数据丢失
- 事务

分布式的应用的结构是下面这样的：

![img](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/v2-8a8fd9da3b83b31a331105ce3b81f75b_1440w.png)

展开来说还有很多，服务架构，自动化运维，自动化部署，版本控制、前端，接口设计等，不过我认为到这里，作为后端的基本职责就算是完成了。除了基本功，帮你走得更远的是内功：**操作系统、数据结构、计算机网络、设计模式、数据库**这些能力能帮助你设计出更加完好的程序。
