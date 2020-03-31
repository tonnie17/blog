+++
title = "django中的session实现"
date = 2018-02-28T00:00:00+08:00
tags = ["python", "django", "session"]
categories = [""]
draft = false
+++

## Cookie

要理解session，首先要搞清cookie的概念。由于http是无状态的，服务器不能“记住”用户的信息状态，因此若同一个客户端发起的多条请求，服务器不能辨别这些请求来自哪个用户，http无状态的限制为web应用程序的设计带来了许多不便，购物网站中的“购物车”功能就是一个很好的例子，当用户把商品放进购物车后，客户端必须要保存购物车的状态，否则当用户下次浏览网站时，购物车拥有的商品状态便不复存在。客户端和服务器必须有通信的媒介，方便服务器追踪客户端的状态，于是Cookie技术应运而生，cookie是服务器产生的一段随机的字符串，发送给客户端，随后客户端便保存cookie，并使用这个cookie附带进后续的请求。以下是cookie设置的详细流程：

1. 客户端发起一个请求连接（如HTTP GET）。
2. 服务器在http响应头上加上`Set-Cookie`，里面存放字符串的键值对。
3. 客户端随后的http请求头加上`Cookie`首部，它包含了之前服务器响应中设置cookie的信息。

根据这个`Cookie`首部的信息，服务器便能“记住”当前用户的信息。

**再来看看在Python中是如何设置Cookie的：**

```python
from BaseHTTPServer import HTTPServer
from SimpleHTTPServer import SimpleHTTPRequestHandler
import Cookie

class MyRequestHandler(SimpleHTTPRequestHandler):
    def do_GET(self):
        content = "<html><body>Path is: %s</body></html>" % self.path
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.send_header('Content-length', str(len(content)))

        cookie = Cookie.SimpleCookie()
        cookie['id'] = 'some_value_42'

        self.wfile.write(cookie.output())
        self.wfile.write('\r\n')

        self.end_headers()
        self.wfile.write(content)

server = HTTPServer(('', 59900), MyRequestHandler)
server.serve_forever()
```

查看服务器端的http响应头，会发现以下字段：

```shell
Set-Cookie: id=some_value_42
```

**而在Django中，可以用如下的方式获取或设置Cookie：**

```python
def test_cookie(request):
    if 'id' in request.COOKIES:
        cookie_id = request.COOKIES['id']
        return HttpResponse('Got cookie with id=%s' % cookie_id)
    else:
        resp = HttpResponse('No id cookie! Sending cookie to client')
        resp.set_cookie('id', 'some_value_99')
        return resp
```

Django通过一系列的包装使得封装Cookie的操作变得更加简单，那么它在其中是怎么实现cookie的读取的呢，下面来窥探原理。

```python
def _get_cookies(self):
    if not hasattr(self, '_cookies'):
        self._cookies = http.parse_cookie(self.environ.get('HTTP_COOKIE', ''))
    return self._cookies
```

可以看出，获取cookie的操作用了Lazy initialization（延迟加载）的技术，因为如果客户端不需要用到cookie，这个过程只会浪费不必要的操作。

再来看`parse_cookie`的实现：

```python
def parse_cookie(cookie):
    if cookie == '':
        return {}
    if not isinstance(cookie, Cookie.BaseCookie):
        try:
            c = SimpleCookie()
            c.load(cookie, ignore_parse_errors=True)
        except Cookie.CookieError:
            # 无效cookie
            return {}
    else:
        c = cookie
    cookiedict = {}
    for key in c.keys():
        cookiedict[key] = c.get(key).value
    return cookiedict
```

它负责解析Cookie并把结果集成到一个dict（字典）对象中，并返回字典。而设置cookie的操作则会被WSGIHandler执行。

注：Django的底层实现了WSGI的接口（如WSGIRequest，WSGIServer等）。

## Session

前面介绍了Cookie的作用，有了Cookie，为什么还需要Session呢？其实很多情况下，只使用Cookie便能完成大部分目的。但是人们发现，只使用Cookie往往是不够的，考虑用户登录信息或一些重要的敏感信息，用Cookie存储的话会带来一些问题，最明显的是由于Cookie会把信息保存到本地，因此信息的安全性可能受到威胁。Session的出现很好地解决的这个问题，Session与Cookie类似，但它们最明显的区别是，Session会将信息保存服务器端，客户端需要一个`session_id`，它一段随机的字符串，类似身份证的功能，从服务器端中根据这个凭证来获取信息。而这个`session_id`通常是保存在Cookie中的，换句话说，Session的信息传递一般要借用到Cookie，如果Cookie被禁用，它则可能通过为url加上query string来添加`session_id`。

下面看一个简单的session应用例子：

```python
def test_count_session(request):
    if 'count' in request.session:
        request.session['count'] += 1
        return HttpResponse('new count=%s' % request.session['count'])
    else:
        request.session['count'] = 1
        return HttpResponse('No count in session. Setting to 1')
```

它用session实现了一个计数器，当每一个请求到来时，就为计数器加一，把新的结果更新到session中。

查看http的响应头，会得到类似下面的信息。

```shell
Set-Cookie:sessionid=a92d67e44a9b92d7dafca67e507985c0;
           expires=Thu, 07-Jul-2011 04:16:28 GMT;
           Max-Age=1209600;
           Path=/
```

里面包含了session_id以及过期时间等信息。

那么服务器端是如何保存session的呢？

在django中，默认会把session保存在setting指定的数据库中，除此之外，也可以通过指定session engine，使session保存在文件(file)，内存(cache)中。

如果保存在数据库中，django会在数据库中创建一个如下的session表。

```sql
CREATE TABLE "django_session" (
    "session_key" varchar(40) NOT NULL PRIMARY KEY,
    "session_data" text NOT NULL,
    "expire_date" datetime NOT NULL
);
```

session_key是放置在cookie中的id，它是唯一的，而session_data则存放序列化后的session数据字符串。

通过`session_key`可以在数据库中取得这条session的信息：

```python
from django.contrib.sessions.models import Session
#...
sess = Session.objects.get(pk='a92d67e44a9b92d7dafca67e507985c0')
print(sess.session_data)
print(sess.get_decoded())
```

输出

```shell
ZmEyNDVhNTBhMTk2ZmRjNzVlYzQ4NTFjZDk2Y2UwODc3YmVjNWVjZjqAAn1xAVUFY291bnRxAksG
cy4=

{'count': 6}
```

回看第一个例子，我们是通过`request.session`来获取session的，为什么请求对象会附带一个session对象呢，这其中做了什么呢？

这就引出了下面要说的django里的中间件技术。

## Session middleware

关于中间件，`<<the Django Book>>`是这样解释的：

> Django的中间件框架，是django处理请求和响应的一套钩子函数的集合。

我们看传统的django视图模式一般是这样的：`http请求->view->http响应`，而加入中间件框架后，则变为：`http请求->中间件处理->app->中间件处理->http响应`。而在django中这两个处理分别对应`process_request`和`process_response`函数，这两个钩子函数将会在特定的时候被触发。

直接看`SessionMiddleware`可能更清晰一些：

```python
class SessionMiddleware(object):
    def __init__(self):
        engine = import_module(settings.SESSION_ENGINE)
        self.SessionStore = engine.SessionStore

    def process_request(self, request):
        session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME)
        request.session = self.SessionStore(session_key)

    def process_response(self, request, response):
        """
        If request.session was modified, or if the configuration is to save the
        session every time, save the changes and set a session cookie or delete
        the session cookie if the session has been emptied.
        """
        try:
            accessed = request.session.accessed
            modified = request.session.modified
            empty = request.session.is_empty()
        except AttributeError:
            pass
        else:
            # First check if we need to delete this cookie.
            # The session should be deleted only if the session is entirely empty
            if settings.SESSION_COOKIE_NAME in request.COOKIES and empty:
                response.delete_cookie(settings.SESSION_COOKIE_NAME,
                    domain=settings.SESSION_COOKIE_DOMAIN)
            else:
                if accessed:
                    patch_vary_headers(response, ('Cookie',))
                if (modified or settings.SESSION_SAVE_EVERY_REQUEST) and not empty:
                    if request.session.get_expire_at_browser_close():
                        max_age = None
                        expires = None
                    else:
                        max_age = request.session.get_expiry_age()
                        expires_time = time.time() + max_age
                        expires = cookie_date(expires_time)
                    # Save the session data and refresh the client cookie.
                    # Skip session save for 500 responses, refs #3881.
                    if response.status_code != 500:
                        try:
                            request.session.save()
                        except UpdateError:
                            # The user is now logged out; redirecting to same
                            # page will result in a redirect to the login page
                            # if required.
                            return redirect(request.path)
                        response.set_cookie(settings.SESSION_COOKIE_NAME,
                                request.session.session_key, max_age=max_age,
                                expires=expires, domain=settings.SESSION_COOKIE_DOMAIN,
                                path=settings.SESSION_COOKIE_PATH,
                                secure=settings.SESSION_COOKIE_SECURE or None,
                                httponly=settings.SESSION_COOKIE_HTTPONLY or None)
        return response
```

在请求到来后，`SessionMiddleware`的`process_request`在请求取出session_key，并把一个新的session对象赋给request.session，而在返回响应时，`process_response`则判断session是否被修改或过期，来更新session的信息。

## 用户认证中的Session

在django中，用下面的方法来验证用户是否登录是常见的事情。

```python
def test_user(request):
    user_str = str(request.user)
    if request.user.is_authenticated():
        return HttpResponse('%s is logged in' % user_str)
    else:
        return HttpResponse('%s is not logged in' % user_str)
```

其实`request.user`的实现也借助到了session。

在这个例子中，成功登录后，session表会保存类似下面的信息，里面记录了用户的id，以后进行验证时，便会到这个表中获取用户的信息。

```shell
{'_auth_user_id': 1, '_auth_user_backend': 'django.contrib.auth.backends.ModelBackend'}
```

跟上面提到的Session中间件相似，用户验证也有一个中间件：AuthenticationMiddleware，在`process_request`中，通过`request.__class__.user = LazyUser()`在request设置了一个全局的可缓存的用户对象。

```python
class LazyUser(object):
    def __get__(self, request, obj_type=None):
        if not hasattr(request, '_cached_user'):
            from django.contrib.auth import get_user
            request._cached_user = get_user(request)
        return request._cached_user

class AuthenticationMiddleware(object):
    def process_request(self, request):
        request.__class__.user = LazyUser()
        return None
```

在get_user里，会在检查session中是否存放了当前用户对应的user_id，如果有，则通过id在model查找相应的用户返回，否则返回一个匿名的用户对象(`AnonymousUser`)。

```python
def get_user(request):
    from django.contrib.auth.models import AnonymousUser
    try:
        user_id = request.session[SESSION_KEY]
        backend_path = request.session[BACKEND_SESSION_KEY]
        backend = load_backend(backend_path)
        user = backend.get_user(user_id) or AnonymousUser()
    except KeyError:
        user = AnonymousUser()
    return user
```

## Django中的Session实现

Django使用的Session默认都继承于`SessionBase`类里，这个类实现了一些session操作方法，以及hash，decode，encode等方法。

```python
class SessionBase(object):
    """
    Base class for all Session classes.
    """
    TEST_COOKIE_NAME = 'testcookie'
    TEST_COOKIE_VALUE = 'worked'

    def __init__(self, session_key=None):
        self._session_key = session_key
        self.accessed = False
        self.modified = False
        self.serializer = import_string(settings.SESSION_SERIALIZER)
```

说的更直白一些，其实django中的session就是一个模拟dict的对象，并实现了一系列的hash和序列化方法，默认持久化在数据库中（有时候也可能由于为了提高性能，用redis之类的内存数据库来缓存session）。
