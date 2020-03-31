+++
title = "Play Python"
date = 2017-03-09T00:00:00+08:00
tags = ["python"]
categories = [""]
draft = false
+++

Hello World

```python
In [1]: import __hello__
Hello world!
```

查看API手册

```shell
~ python
11/17 17:02:27 2016
Python 2.7.12 (default, Jun 29 2016, 14:05:02)
[GCC 4.2.1 Compatible Apple LLVM 7.3.0 (clang-703.0.31)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> help()

Welcome to Python 2.7!  This is the online help utility.

If this is your first time using Python, you should definitely check out
the tutorial on the Internet at http://docs.python.org/2.7/tutorial/.

Enter the name of any module, keyword, or topic to get help on writing
Python programs and using Python modules.  To quit this help utility and
return to the interpreter, just type "quit".

To get a list of available modules, keywords, or topics, type "modules",
"keywords", or "topics".  Each module also comes with a one-line summary
of what it does; to list the modules whose summaries contain a given word
such as "spam", type "modules spam".

help> os.path

help>

--------- ipython ----------

In [1]: import os

In [2]: os.path?
Type:        module
String form: <module 'posixpath' from '/usr/local/Cellar/python3/3.6.0/Frameworks/Python.framework/Versions/3.6/lib/python3.6/posixpath.py'>
File:        /usr/local/Cellar/python3/3.6.0/Frameworks/Python.framework/Versions/3.6/lib/python3.6/posixpath.py
Docstring:
Common operations on Posix pathnames.

Instead of importing this module directly, import os and refer to
this module as os.path.  The "os.path" name is an alias for this
module on Posix systems; on other systems (e.g. Mac, Windows),
os.path provides the same operations in a manner specific to that
platform, and is an alias to another module (e.g. macpath, ntpath).

Some of this can actually be useful on non-Posix systems too, e.g.
for manipulation of the pathname component of URLs.

In [3]: os.path??
```

遍历列表

```python
array = [1, 2, 3, 4, 5]

for i in range(len(array)):
    print (i, array[i])
    
# Pythonic 
for i, item in enumerate(array):
    print (i, item)
```

对列表的操作

```python
array = [1, 2, 3, 4, 5]

new_array = []
for item in array:
    new_array.append(str(item))

# Pythonic
new_array = [str(item) for item in array]

# Generator
new_array = (str(item) for item in array)

# 函数式
new_array = map(str, array)
```

列表推导

```python
# 生成列表
[ i*i for i in range(10) if i % 2 == 0 ]
# 生成集合
{ i*i for i in range(10) if i % 2 == 0 }
# 生成字典
{ i:i for i in range(10) if i % 2 == 0 }
```

上下文管理器

```python
file = open('file', 'w')
file.write(123)
file.close()

# Pythonic
with open('file', 'w') as file:
    file.write(123)
```

条件判断

```python
if x is True:
    y = 1
else:
    y = -1

# Pythonic
y = 1 if x is True else -1
```

构造矩阵

```python
y = [0 for _ in range(100000000)]
# Pythonic
y = [0] * 100000000
```

装饰器

```python
def logic(x):
    if x < 0:
        return False
    print (x)
    return True

#Pythonic
def check_gt_zero(func):
    def wrapper(x):
        if x < 0:
            return False
        return func(x)
    return wrapper 

@check_gt_zero
def logic(x):
    print (x)
    return True
```

变量交换

```python
temp = y
y = x
x = temp

# Pythonic
x, y = y, x
```

切片

```python
array = [1, 2, 3, 4, 5]

l = len(array)
for i in range(l/2):
    temp = array[l - i - 1]
    array[l - i - 1] = array[i]
    array[i] = temp

array = list(reversed(array))
# Pythonic
array = array[::-1]
```

读取文件

```python
CHUNK_SIZE = 1024

with open('test.json') as f:
    chunk = f.read(CHUNK_SIZE)
    while chunk:
        if chunk:
            print(chunk)
        chunk = f.read(CHUNK_SIZE)

from functools import partial
# Pythonic
with open('test.json') as f:
    for piece in iter(partial(f.read, CHUNK_SIZE), ''):
        print (piece)
# Lambda
with open('test.json') as f:
    for piece in iter(lambda: f.read(CHUNK_SIZE), ''):
        print (piece)
```

for-else和try-else语法

```python
is_for_finished = True

try:
    for item in array:
        print (item)
        # raise Exception
except:
    is_for_finished = False

if is_for_finished is True:
    print ('complete')
    
# Pythonic
for item in array:
    print (item)
    # raise Exception
else:
    print ('complete')

try:
    print ('try')
    # raise Exception
except Exception:
    print ('exception')
else:
    print ('complete')
```

函数参数解压

```python
def draw_point(x, y):
    # do some magic

point_foo = (3, 4)
point_bar = {'y': 3, 'x': 2}

draw_point(*point_foo)
draw_point(**point_bar)
```

列表/元组解压

```python
first, second, *rest = (1,2,3,4,5,6,7,8)
```

"Print To"语法

```python
# 2.x
print >>  open("myfile", "w"), "hello world"
# 3.x
print ("hello world", file=open("myfile", "w"))
```

字典缺省值

```python
d = {}

try:
    d['count'] = d['count'] + 1
except KeyError:
    d['count'] = 0
# Pythonic
d['count'] = d.get('count', 0) + 1
```

链式比较符

```python
if x < 100 and x > 0:
    print(x)
# Pythonic
if 0 < x < 100:
    print(x)
```

多行字符串

```python
s = ("longlongstringiii"
    "iiiiiiiii"
    "iiiiiii")

print(s)
```

in表达式

```python
if 'string'.find('ring') > 0:
    print ('find')
# Pythonic
if 'ring' in 'string':
    print ('find')

for r in ['ring', 'ring1', 'ring2']:
    if r == 'ring':
        print ('find')
# Pythonic
if 'ring' in ['ring', 'ring1', 'ring2']:
    print('find')
```

字符串连接

```python
array = ['a', 'b', 'c', 'd', 'e']
s = array[0]
for char in array[1:]:
    s += ',' + char
# Pythonic
s = ','.join(array)
```

列表合并字典

```python
keys = ['a', 'b', 'c']
values = [1, 2, 3]

d = {}
for key, value in zip(keys, values):
    d[key] = value
# Better
d = dict(zip(keys, values))
# Pythonic
d = {key: value for key, value in zip(keys, values)}
```

all和any

```python
flag = True
for cond in conditions:
    if cond is False:
        flag = False
        break
# Pythonic
flag = all(conditions)

flag = False
for cond in conditions:
    if cond is True:
        flag = True
        break
# Pythonic
flag = any(conditions)
```

or

```python
not_null_string = '' or 'string1' or 'string2'
# string1
```

方法提取

```python
for item in ['b', 'c']:
    array.append(item)
# Faster
append = array.append
for item in ['b', 'c']:
    append(item)
```

单例模式

```python
from [module] import [single_instance]
```

让python的语法像ruby一样

```python
__builtins__.end = None

def ruby_like_func(x):
    print(x)
end
```

用<>代替!=

```python
In [1]: from __future__ import barry_as_FLUFL

In [2]: 1 == 1
Out[2]: True

In [3]: 1 != 1
  File "<ipython-input-3-1ec724d2c3b1>", line 1
    1 != 1
       ^
SyntaxError: invalid syntax


In [4]: 1 <> 1
Out[4]: False
```

性能排查

```shell
python -m cProfile test.py
```

格式化一个json文件

```shell
python -m json.tool test.json
echo '{"json":"obj"}' | python -m json.tool # 管道
```

开启本地目录的HTTP服务器

```shell
python2 -m SimpleHTTPServer 8080 # 2.x
python3 -m http.server 8080 # 3.x
```

在Python使用花括号块

```python
from __future__ import braces
```

Python之禅

```shell
In [1]: import this
The Zen of Python, by Tim Peters

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

antigravity

```python
import antigravity
```

