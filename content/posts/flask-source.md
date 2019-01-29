+++
title = "Flask源码阅读笔记"
date = 2015-12-01T00:00:00+08:00
tags = ["python"]
categories = [""]
draft = false
+++

首先看一个来自官方的使用例子：

```Python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run()
```

那么这里面发生了什么呢？进入源码一探究竟。

这是Flask的依赖的模块，可以看到Flask依赖了`Jinja2`以及`Werkzeug`，Flask中关于模板大部分是封装了`jinja template method`的一些通用方法，包括常用的`render_template`，`Werkzeug`是Python中一个关于WSGI协议的工具库，支撑了Flask这个框架的大多数功能，例如：请求（`Request`）和响应（`Response`）对象，`LocalStack`和`LocalProxy`则是使用了本地线程的代理对象，`Map`和`Rule`则负责了定义路由的模版规则和映射方法，`SecureCookie`则是`Session`对象要继承的父类，它包含了处理cookie的一些机制和方法。除此之外，还导入了一些中间件（`SharedDataMiddleware`），一些辅助函数（`create_environ`,`wrap_file`，`cached_property`），异常类（`HTTPException`,`InternalServerError`）等。

```python
//...
from jinja2 import Environment, PackageLoader, FileSystemLoader
from werkzeug import Request as RequestBase, Response as ResponseBase, \
     LocalStack, LocalProxy, create_environ, SharedDataMiddleware, \
     ImmutableDict, cached_property, wrap_file, Headers, \
     import_string
from werkzeug.routing import Map, Rule
from werkzeug.exceptions import HTTPException, InternalServerError
from werkzeug.contrib.securecookie import SecureCookie
```

首先看`Flask`类被初始化的时候做了什么事：

- 初始化了一个`Config`对象，即应用使用的全局配置，它是一个`dict`对象，不过扩展了各种配置加载的方法（`from_envvar`,`from_pyfile`,`from_object`）。
- 初始化了`view_functions`（注册所有的视图函数，如例子中的`hello`），初始化了`error_handlers`（它负责处理像`404`，`500`这类错误的跳转视图函数）。
- 初始化了`before_request_funcs`和`after_request_funcs`，它们分别对应存储请求进入前和离开后调用的钩子函数的字典对象。
- 初始化了`url_map`，它是Werkzeug中实现的`Map`类，负责做路由的映射。
- 做了静态资源路径的配置以及`jinja`（一个模版引擎）的一些初始化工作。

```python
class Flask(_PackageBoundObject):
    request_class = Request

    response_class = Response

    static_path = '/static'

    debug = ConfigAttribute('DEBUG')

    secret_key = ConfigAttribute('SECRET_KEY')

    session_cookie_name = ConfigAttribute('SESSION_COOKIE_NAME')

    permanent_session_lifetime = ConfigAttribute('PERMANENT_SESSION_LIFETIME')

    use_x_sendfile = ConfigAttribute('USE_X_SENDFILE')

    debug_log_format = (
        '-' * 80 + '\n' +
        '%(levelname)s in %(module)s, %(pathname)s:%(lineno)d]:\n' +
        '%(message)s\n' +
        '-' * 80
    )

    jinja_options = ImmutableDict(
        autoescape=True,
        extensions=['jinja2.ext.autoescape', 'jinja2.ext.with_']
    )

    default_config = ImmutableDict({
        'DEBUG':                                False,
        'SECRET_KEY':                           None,
        'SESSION_COOKIE_NAME':                  'session',
        'PERMANENT_SESSION_LIFETIME':           timedelta(days=31),
        'USE_X_SENDFILE':                       False
    })

    def __init__(self, import_name):
        _PackageBoundObject.__init__(self, import_name)

        self.config = Config(self.root_path, self.default_config)
        self.view_functions = {}
        self.error_handlers = {}
        self.before_request_funcs = {}
        self.after_request_funcs = {}
        self.template_context_processors = {
            None: [_default_template_ctx_processor]
        }

        self.url_map = Map()

        if self.static_path is not None:
            self.add_url_rule(self.static_path + '/<filename>',
                              build_only=True, endpoint='static')
            if pkg_resources is not None:
                target = (self.import_name, 'static')
            else:
                target = os.path.join(self.root_path, 'static')
            self.wsgi_app = SharedDataMiddleware(self.wsgi_app, {
                self.static_path: target
            })

        self.jinja_env = Environment(loader=self.create_jinja_loader(),
                                     **self.jinja_options)
        self.jinja_env.globals.update(
            url_for=url_for,
            get_flashed_messages=get_flashed_messages
        )
        self.jinja_env.filters['tojson'] = _tojson_filter
```

在调用`@app.route`装饰器时做了什么呢？其实就是往`url_map`里面添加了一条规则，包含了诸如endpoint，method等信息，最后会把endpoint通过`view_functions`绑定到具体函数（`hello`)。

```python
def route(self, rule, **options):
    def decorator(f):
        self.add_url_rule(rule, None, f, **options)
        return f
    return decorator

def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
    if endpoint is None:
        assert view_func is not None, 'expected view func if endpoint ' \
        'is not provided.'
    endpoint = view_func.__name__
    options['endpoint'] = endpoint
    options.setdefault('methods', ('GET',))
    self.url_map.add(Rule(rule, **options))
    if view_func is not None:
        self.view_functions[endpoint] = view_func
```

接着看`run`方法的实现，它就是把`Flask`类实例当成了一个`wsgi`对象传给Werkzeug，再借由Werkzeug来运行web服务。

```python
def run(self, host='127.0.0.1', port=5000, **options):
    from werkzeug import run_simple
    if 'debug' in options:
        self.debug = options.pop('debug')
    options.setdefault('use_reloader', self.debug)
    options.setdefault('use_debugger', self.debug)
    return run_simple(host, port, self, **options)
```

`Flask`类也体现了实现wsgi协议的地方，`wsgi_app`方法就是wsgi的application对象，在`__call__`方法中返回了这个方法的调用，因此我们在使用一些WSGI服务器（如wsgiref，gunicorn，uwsgi等）时，可以把app的实例传递给web server。

```python
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):
        try:
            rv = self.preprocess_request()
            if rv is None:
                rv = self.dispatch_request()
            response = self.make_response(rv)
            response = self.process_response(response)
        except Exception, e:
            response = self.make_response(self.handle_exception(e))
        return response(environ, start_response)

def __call__(self, environ, start_response):
    """Shortcut for :attr:`wsgi_app`."""
    return self.wsgi_app(environ, start_response)
```

通过之前可以看到，`Flask`类继承了一个Mixin类：`_PackageBoundObject`，`_PackageBoundObject`作为包装一个模块或包的`Mixin`，无论是app还是module，它都会继承这个`_PackageBoundObject`。在0.3这个版本虽然`BluePrint`还没出现，但已经有了多模块的概念，例如下面要提到的`Module`类，我们可以在一个app中定义`admin`和`main`两个模块，那么这里的`admin`和`main`都是被`_PackageBoundObject`包装过的对象，里面包含了模块的导入名称(`import_name`，在register的时候用）和根路径（`root_path`，定义了寻找这个模块的路径）。

`_ModuleSetupState`则是模块被装载时，用来包装模块信息的对象，与`Flask`类中的`register_module`具有强烈关联。

```python
class _PackageBoundObject(object):

    def __init__(self, import_name):
        #: the name of the package or module.  Do not change this once
        #: it was set by the constructor.
        self.import_name = import_name

        #: where is the app root located?
        self.root_path = _get_package_path(self.import_name)

    def open_resource(self, resource):
        if pkg_resources is None:
            return open(os.path.join(self.root_path, resource), 'rb')
        return pkg_resources.resource_stream(self.import_name, resource)
 

class _ModuleSetupState(object):

    def __init__(self, app, url_prefix=None):
        self.app = app
        self.url_prefix = url_prefix
```

Flask中的`Module`类同样继承了`_PackageBoundObject`，`Module`作为Flask模块化设计的一部分可以作为一个子模块被flask app注册进去，`Module`包含了一些基本信息：`name`(模块名称)，`url_prefix`（路由地址），`_register_events`（事件函数），`route`，`add_url_rule`这两个方法提供了添加视图函数路由的方式，使用过flask的都不会陌生，特别是`@app.route('/')`这种装饰方法，就是在这里来实现，最终都会调用`add_url_rule`这个方法，注意里面中`state.app.add_url_rule`的这条语句，这个state就是上面说的`_ModuleSetupState`，实际上，flask app在注册模块时会调用模块中`_register_events`中的所有事件，仔细看`Module`中的`add_url_rule`方法，它会把`register_rule`这个事件放在`_register_events`，想知道为什么这么做可直接看`Flask`类中`register_module`方法，其实app在`register_module`时，会生成一个state(`_ModuleSetupState`)，它包含了flask app自身的实例，然后遍历`_register_events`，把state作为参数，调用所有事件函数！因此我们再来看`Module`中的`add_url_rule`方法里面的`state.app.add_url_rule`，其实它的作用就是把路由注册到当前模块所在flask app的`url_map`中，可以看到它会在路由的前面加上模块自身的`name`。同理，下面的`before_request`，`before_app_request`，`after_request`，`after_app_request`等等的hook都是返回一个需要**state**作为参数的event function。也就是说，在分发的模块上做的工作最终都会在flask app中得到实现。

```python
class Module(_PackageBoundObject):

    def __init__(self, import_name, name=None, url_prefix=None):
        if name is None:
            assert '.' in import_name, 'name required if package name ' \
                'does not point to a submodule'
            name = import_name.rsplit('.', 1)[1]
        _PackageBoundObject.__init__(self, import_name)
        self.name = name
        self.url_prefix = url_prefix
        self._register_events = []

    def route(self, rule, **options):
        def decorator(f):
            self.add_url_rule(rule, f.__name__, f, **options)
            return f
        return decorator

    def add_url_rule(self, rule, endpoint, view_func=None, **options):
        def register_rule(state):
            the_rule = rule
            if state.url_prefix:
                the_rule = state.url_prefix + rule
            state.app.add_url_rule(the_rule, '%s.%s' % (self.name, endpoint),
                                   view_func, **options)
        self._record(register_rule)

    def before_request(self, f):
        self._record(lambda s: s.app.before_request_funcs
            .setdefault(self.name, []).append(f))
        return f

    def before_app_request(self, f):
        self._record(lambda s: s.app.before_request_funcs
            .setdefault(None, []).append(f))
        return f

    def after_request(self, f):
        self._record(lambda s: s.app.after_request_funcs
            .setdefault(self.name, []).append(f))
        return f

    def after_app_request(self, f):
        self._record(lambda s: s.app.after_request_funcs
            .setdefault(None, []).append(f))
        return f

    def context_processor(self, f):
        self._record(lambda s: s.app.template_context_processors
            .setdefault(self.name, []).append(f))
        return f

    def app_context_processor(self, f):
        self._record(lambda s: s.app.template_context_processors
            .setdefault(None, []).append(f))
        return f

    def _record(self, func):
        self._register_events.append(func)
```

关于Flask模块化的机制，现在应该清楚了，接着再来探讨Flask是怎么处理每个到来的请求的呢？

Flask中定义了`Request`和`Response`类，分别对应请求和响应对象，它们继承了Werkzeug的`Request`和`Response`对象，它们已经帮你封装好了大部分东西（例如http头部的解析，environ，path_info，json，异常处理等）。

`_RequestGlobals`是关于某个请求中全局对象的一个集合，也就是flask中的g对象。

```python
class Request(RequestBase):

    endpoint = view_args = routing_exception = None

    @property
    def module(self):
        if self.endpoint and '.' in self.endpoint:
            return self.endpoint.rsplit('.', 1)[0]

    @cached_property
    def json(self):
        if __debug__:
            _assert_have_json()
        if self.mimetype == 'application/json':
            return json.loads(self.data)


class Response(ResponseBase):
    default_mimetype = 'text/html'


class _RequestGlobals(object):
    pass
```

用过Flask的同学都知道，在视图函数想要获取当前请求的时候一般会通过`from flask import request`，然后在任何地方使用这个request对象，十分方便。

那么这种获取请求的机制是如何实现的呢？看Flask源码我们可以看到它定义了几个全局对象：

```python
# context locals
_request_ctx_stack = LocalStack()
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)
request = LocalProxy(lambda: _request_ctx_stack.top.request)
session = LocalProxy(lambda: _request_ctx_stack.top.session)
g = LocalProxy(lambda: _request_ctx_stack.top.g)
```

它们都是Flask提供的ThreadLocal对象：

- `_request_ctx_stack`：保存请求上下文对象的栈（下面的栈表示这个）。
- `current_app`：栈顶请求（即当前请求）所在的应用（app）的代理对象。
- `request`：栈顶请求的代理对象。
- `session`：栈顶请求的会话的代理对象。
- `g`：栈顶请求的全局对象的代理对象。

可以看到`request`对象是一个请求上下文对象的代理，`_request_ctx_stack`其实一个维护`_RequestContext`的栈，那么它的作用是什么呢？其实，每当flask接受一个`request`之后，它就会把它和一些信息封装成一个`_RequestContext`的实例，然后把这个请求上下文塞到`_request_ctx_stack`栈中，当请求结束后，这个栈就会把`_RequestContext`弹出，很明显，这个`_request_ctx_stack`就是存储请求上下文的栈，除此之外，它还是一个`LocalStack`，即用本地线程（`ThreadLocal`）实现的栈，它保证了请求对象(**全局对象**)在多线程环境中是*线程隔离的*，在这里你也可以观察到flask与django中不同的地方，即django中写一个视图函数通常要带一个`request`参数，而flask则不用，原因就在这里。

那么`_RequestContext`又是什么？`_RequestContext`是flask中一个非常重要的概念，它的语义即关于**请求的上下文**，它里面封装了关于请求的所有相关信息，例如：

- `app`：请求所在的应用(*flask是多应用机制*)。
- `url_adapter`：负责url mapping的适配器，关联的是Werkzeug的`Map`。
- `request`：当前的请求对象。
- `session`：当前`request`的会话对象。
- `g`：即前面的\_RequestGlobal，它存储了关于当前请求的全局对象，例如我们在做用户登录的时候通常会用到它来存储user对象。
- `flashes`：关于当前请求的*闪现信息(flash message)*。

```python
class _RequestContext(object):

    def __init__(self, app, environ):
        self.app = app
        self.url_adapter = app.url_map.bind_to_environ(environ)
        self.request = app.request_class(environ)
        self.session = app.open_session(self.request)
        if self.session is None:
            self.session = _NullSession()
        self.g = _RequestGlobals()
        self.flashes = None

        try:
            self.request.endpoint, self.request.view_args = \
                self.url_adapter.match()
        except HTTPException, e:
            self.request.routing_exception = e

    def push(self):
        _request_ctx_stack.push(self)

    def pop(self):
        _request_ctx_stack.pop()

    def __enter__(self):
        self.push()
        return self

    def __exit__(self, exc_type, exc_value, tb):
        if tb is None or not self.app.debug:
            self.pop()
```

其他诸如`flash`，`url_for`，`Session`等有趣的功能也不单独说了，关于Flask的主要实现到这里基本就结束了。

